> How many people chat with Gemini every day?

---

Based on available estimates and reported data from 2025, the number of people who chat with Gemini every day is estimated to be around **35 million to 45 million Daily Active Users (DAU)**.

Here are a few specific data points that have been reported:

* **35 million DAU** as of March 2025.
* **40-45 million DAU** as of May 2025 (estimated).

It's important to note that these figures are often estimates from third parties or internal company data revealed in specific contexts, and the number is likely growing. Also, this count generally refers to direct interaction with the Gemini chatbot on the web or mobile app, and may not include all users who benefit from Gemini's underlying technology integrated into other Google products like Search's AI Overviews, which reach a much larger audience.

---

> I want to create a customization of TEI. I like TEI exactly as it is except for one thing. I would like any element to have the option to be empty (minOccurs="0"), or to minimize content so as to not include subelements that would have been mandatory (minOccurs="1"). How would you suggest I go about doing this?

---

That's an interesting customization request. You're essentially asking for a way to override the mandatory content model (i.e., `minOccurs="1"` for elements or mandatory subelements) for *any* TEI element, allowing it to be either empty or to contain only minimal content, while keeping the rest of the TEI schema intact.

The most practical and sustainable way to achieve this within the TEI customization framework is by using a combination of **ODD (One Document Does it All)** customization and the **TEI processing model**.

-----

## 1\. The Strategy: Custom `data.content` and `content` Attributes

Instead of modifying the content models of hundreds of elements, which would be error-prone and hard to maintain, you should use attributes to signal that the element's content has been deliberately suppressed. You would introduce a new attribute class that can be attached to all relevant elements.

### A. Define a New Attribute Class

Create a new attribute class, for example, `att.laxContent`, which includes your special attribute(s).

| Attribute | Description | Values |
| :--- | :--- | :--- |
| **@lax** | A flag to indicate that the content model is intentionally *not* being followed, allowing the element to be empty or contain minimal content. | `"empty"` or `"minimal"` |

### B. ODD Customization

You will need to create an ODD file to define this class and attach it.

```xml
<schemaSpec ident="myLaxTEI" start="TEI" source="tei:TEI">
  <moduleRef key="tei"/>
  <classSpec ident="att.laxContent" type="atts">
    <attList>
      <attDef ident="lax" usage="opt">
        <datatype maxOccurs="unbounded">
          <dataRef key="teidata.enumerated"/>
        </datatype>
        <valList type="closed">
          <valItem ident="empty"><desc>The element must be empty, overriding any mandatory content model.</desc></valItem>
          <valItem ident="minimal"><desc>The element must contain minimal content, suppressing mandatory subelements.</desc></valItem>
        </valList>
      </attDef>
    </attList>
  </classSpec>

  <classRef key="att.global.atts" module="core" mode="change">
    <moduleRef key="myLaxContent"/>
  </classRef>

  <moduleRef key="custom"/>
  </schemaSpec>
```

-----

## 2\. Validation and Processing

The TEI ODD framework is primarily for **generating a schema (like Relax NG or W3C Schema)** for validation. However, the schema itself **cannot conditionally change the content model based on an attribute value** (e.g., if `@lax` is present, ignore `minOccurs="1"`).

Therefore, this solution requires a **two-pronged approach**:

### A. Schema Validation (The Technical Compromise)

You must accept that the generated schema **will still enforce the original TEI content model**.

  * If an element like `<titleStmt>` normally requires a mandatory `<title>` and you mark it `@lax="empty"`, a standard XML validator **will still throw an error**.

The best way to handle this during the **schema generation phase** is to decide:

1.  **Keep the original TEI schema (Recommended):** The generated schema validates *against* the base TEI. You use the `@lax` attribute purely as a **flag** to your post-processing/rendering tools, accepting that your XML files will contain **validation errors** according to the schema but will be **compliant with your institutional policy**.
2.  **Generate a new, looser schema (Harder):** You would have to explicitly redefine *every* element in your ODD to use a relaxed content model (e.g., changing `(foo, bar)` to `(foo?, bar?)` for everything), but this defeats the purpose of "liking TEI exactly as it is" and makes maintenance a nightmare.

**The recommended path is to use the attribute as a policy flag (Option 1).**

### B. Post-Validation / Policy Enforcement (The Real Solution)

The real-world enforcement of your "lax content" policy happens in the **processing pipeline** (e.g., XSLT or XQuery), not the schema validator.

1.  **If `@lax="empty"`:** Your XSLT/XQuery processing scripts would be instructed to treat the element as valid and proceed, knowing the policy was intentionally overridden by the author. When rendering, it would simply render nothing for this element.
2.  **If `@lax="minimal"`:** Your processing scripts would look for an essential text node or minimal subelement (like a `<ref>` or a simple string) and ignore any missing mandatory subelements, again proceeding as valid.

This approach maintains the integrity of the original TEI schema (which is essential for TEI tool compatibility) while giving your documents the structural flexibility you require via an explicit, machine-readable attribute.

---

> Would @lax="minimal" actually suffice either way? If an element is empty, would that still be technically correct with the "minimal" value?

---

That's a very insightful follow-up question that gets to the heart of how you want to define "minimal" content.

The short answer is: **No, the `@lax="minimal"` value would likely not cover the "empty" case, and you would need to use both values or refine your definition of "minimal."**

Here is a breakdown of why, and how to define your attribute values based on your needs:

### 1. The Distinction Between "Minimal" and "Empty"

The original TEI content models are very precise.

* **Empty:** An element with no content at all (no text, no child elements).
* **Minimal:** An element with *some* content, even if it's just a single text node or a single allowed child element, but *not* the full mandatory set defined by the schema.

| Scenario | `@lax` Value that applies | Why |
| :--- | :--- | :--- |
| **Original Problem 1:** Element must be empty (overriding `minOccurs="1"`). | **`@lax="empty"`** | This is explicit and clearly communicates the intent to override the content model entirely, resulting in no content. |
| **Original Problem 2:** Element must have *some* content, but not the mandatory subelements. | **`@lax="minimal"`** | This communicates the intent to provide *just enough* content (e.g., a simple string) while suppressing mandatory complex subelements. |
| **Your Question:** Element is empty, but you use `@lax="minimal"`. | ***Does not suffice.* / Needs clarification.** | In a processing pipeline, "minimal" implies looking for *something*. If the element is truly empty, the pipeline may fail, or it won't be clear if the empty state was intentional or an error. |

### 2. Suggestion: Keep Both Values

To be clear and robust in your policy and processing pipeline, it is best to keep both values as you originally designed them:

* Use **`@lax="empty"`** when you want to signal to your processor: "Ignore the content model; this element must have NO content."
    * *Example:* `<titleStmt lax="empty"/>`
* Use **`@lax="minimal"`** when you want to signal: "Ignore the mandatory subelements; this element must have only a text string or one single, simple subelement (e.g., a simple `<name>`)."
    * *Example:* `<p lax="minimal">A quick note.</p>` (instead of a `<p>` that might mandate a following `<note>` or `<head>`)

### 3. Alternative: Redefine "Minimal" to Include "Empty"

If you absolutely want a single value, you could redefine your single attribute value, say **`@lax="suppressed"`**, with a new policy:

* **Policy:** If the element is marked `@lax="suppressed"`, the processor will validate it against a content model of `((#PCDATA | %phrase.content)*)`. The author is free to make it either empty or give it minimal content.

However, this makes the author's intent less explicit for future maintainers. For the sake of **clarity and machine-readability**, the dual-value approach (`empty` vs. `minimal`) is technically superior for covering your two distinct use cases.
