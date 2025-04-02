# Classification-Agent

##Highlevel Instructions - this is aimed a guide and NOT a detailed walk through. 

# 🧠 Autonomous Document Classification – High-Level Setup Documentation

This guide outlines the high-level steps to set up an AI-powered, automated document classification system using Microsoft Copilot Studio, Power Automate, SharePoint, and Microsoft Purview. It enables documents to be automatically labeled based on their content, with metadata updates and compliance labeling via Purview.

## 🔹 1. General Instructions (Agent Guidance)

Defines how the AI agent interprets content and returns classification.

**Purpose:** The agent determines the appropriate sensitivity label based on document content, following a Knowledge Source containing classification policy.

**Labels:**
- Personal
- Public
- General
- Confidential
- Highly Confidential

**Decision Criteria:**
- Does the content include personal data or regulated information?
- Who is the intended audience?
- Could disclosure cause legal, financial, or reputational harm?

**Expected Output:**
- Classification_Label: One of the labels above.
- Brief_Explanation: A short rationale (1–2 sentences).

**Restrictions:**
- Do not quote internal policy documents.
- Do not speculate or over-explain.
- Do not output uncertain or placeholder values.

## 🔹 2. Action Instructions (HTTP Request to SharePoint)

Defines how and when the agent triggers metadata updates.

**When to Trigger:**
- A classification label has been selected.
- The server-relative file path is available.
- The SharePoint library contains a column called AISensitivity.

**Request Configuration:**
- Method: POST
- URI:
```
_api/web/GetFileByServerRelativeUrl('OriginalDocumentPath')/ListItemAllFields
```
(Note: OriginalDocumentPath must begin with / and be wrapped in single quotes.)

**Headers:**
```
Accept: application/json;odata=verbose
Content-Type: application/json;odata=verbose
IF-MATCH: *
X-HTTP-Method: MERGE
```

**Body:**
```json
{
  "__metadata": { "type": "SP.Data.Shared_x0020_DocumentsItem" },
  "AISensitivity": "Classification_Label"
}
```

## 🔹 3. SharePoint Document Library Setup

Prepares SharePoint to support AI-driven classification.

**Create Custom Column:**
- Column Name: AISensitivity
- Type: Single line of text (or managed metadata, if preferred)

**Permissions:**
- Ensure the Copilot Studio agent (via HTTP connector or service principal) has permission to update metadata in the library.

**Content:**
- Documents must be stored in a library with a predictable path such as /Shared Documents/.

## 🔹 4. Power Automate Flow (Trigger and Processing)

Sets up automation logic to detect changes, classify content, and apply labels.

**Trigger:**
- When a file is created or modified (properties only) (polling-based trigger)
- Frequency: Adjust to hourly or daily depending on volume (default is 1 min)

**Double-Trigger Prevention:**
- Add a condition step at the start:
```
If AISensitivity is empty
```
This ensures files are only processed if they haven't already been classified.

**Expression (Power Automate condition):**
```
@empty(triggerOutputs()?['body/AISensitivity'])
```

**Content Extraction:**
- Primary: Use "Get File Content" and built-in file parsers or OCR.
- Alternative: Use Graph API to retrieve raw document content:
```
GET /drives/{drive-id}/items/{item-id}/content?format={format}
```
Graph Docs →

**Send to GPT Agent:**
- Pass the full text into Copilot Studio with the classification instructions.
- Capture the label and rationale.

**HTTP Request Action:**
- Dynamically inject the label and server-relative path into the HTTP request.
- POST the metadata update back into SharePoint using the action instructions above.

## 🔹 5. Copilot Studio Agent Setup

Configure the classification logic within Microsoft Copilot Studio.

**General Instructions:** Guide how to interpret document content and apply labels.

**Knowledge Source:** Upload a "Document Classification Scheme" file with policy details.

**Agent Outputs:**
- Label (e.g., Confidential)
- Reason for label

**Action Configuration:**
- Trigger HTTP request only when:
  - A label is confidently selected
  - File path is available

## 🔹 6. Microsoft Purview Sensitivity Label Integration

Use Microsoft Purview to enforce real sensitivity labels based on the SharePoint metadata.

**Auto-Label Policy:**
- Create a rule using document property matching:
```
Document property is: AISensitivity:Confidential
```
- Add one rule per classification level if needed.

**Managed Property Mapping:**
- Search Schema: Map the ows_AISensitivity crawled property to a managed property (e.g., RefinableString01).
- Make it queryable, searchable, and retrievable.

**Simulation & Enforcement:**
- Use simulation mode to test matches before publishing auto-labeling policies.
- Publish the policy to activate automatic labeling.

## 🔹 7. Testing & Validation

Ensure the system is working as expected before full rollout.

**Upload sample documents and verify:**
- Flow is triggered correctly.
- Agent returns appropriate classification.
- HTTP request updates SharePoint metadata.
- Microsoft Purview detects and applies correct label via auto-labeling.

## 🔹 8. Optional Enhancements & Governance

- **Audit Logging:** Save label outcomes to a SharePoint list or logging service.
- **Review Loop:** Add a manual override field (e.g., CorrectedSensitivity) to track feedback.
- **Security Separation:** Ensure flow, Copilot, and Purview permissions are scoped properly.
- **File Type Filtering:** Only classify supported formats (e.g., .docx, .pdf).

## ✅ Final Outcome

Documents are automatically:
- Triggered when added or changed
- Read and classified by an AI agent
- Tagged with metadata in SharePoint
- Detected by Purview policies for official sensitivity labeling

This creates a self-sustaining, scalable classification system across Microsoft 365.





======================================================================================

FLOW INSTRUCTIONS

======================================================================================

🚀 Detailed Power Automate Flow Setup Guide
This guide describes how to set up the Power Automate Flow for automatically triggering an autonomous document classification agent.

Step-by-Step Flow Configuration
Step 1: Trigger - SharePoint
Action: "When a file is created or modified (properties only)"

Purpose: Starts the flow when a document is added or updated in SharePoint.

Configuration:

Set your Site Address.

Choose your Document Library.

Step 2: Initialize Variable - FilePathVariable
Action: "Initialize variable"

Purpose: Store the file path for later use.

Configuration:

Name: FilePathVariable

Type: String

Value: Dynamically set from Trigger (file path).

Step 3: Initialize Variable - FileIdentifierVariable
Action: "Initialize variable"

Purpose: Store the file ID or identifier for referencing the document later.

Configuration:

Name: FileIdentifierVariable

Type: String

Value: Dynamically set from Trigger (file identifier).

✅ Extract Document Text Content
Step 4: Get file content
Action: "Get file content" (SharePoint)

Purpose: Retrieve the file content from SharePoint.

Configuration:

Site Address: your SharePoint URL

File Identifier: Use dynamic value from FileIdentifierVariable.

Step 5: Create file (OneDrive)
Action: "Create file" (OneDrive)

Purpose: Temporarily store the document content for text extraction.

Configuration:

Folder Path: e.g., /Temp

File Name: Dynamic (original filename)

File Content: from "Get file content" step.

Step 6: Convert file (OneDrive)
Action: "Convert file"

Purpose: Converts file to a compatible format for OCR (e.g., PDF).

Configuration:

File: Select from "Create file" action.

Target Type: PDF.

Step 7: Delete file (OneDrive)
Action: "Delete file"

Purpose: Removes the temporary file from OneDrive storage after conversion.

Configuration:

File: Original file from "Create file" step.

Step 8: Recognize text in image or PDF (OCR - AI Builder)
Action: "Recognize text in an image or a PDF document"

Purpose: Extract text from PDF document using OCR.

Configuration:

Document type: PDF

File Content: Output from "Convert file" step.

🔹 Call GenAI / GPT for Document Classification
Step 9: Create text with GPT using a prompt
Action: "Create text with GPT using a prompt"

Purpose: Use GPT (via Copilot Studio) to classify document based on extracted text.

Configuration:

Provide clear instructions for GPT to classify the document.

Insert the extracted text as input.

Step 10: Send a prompt to the specified Copilot for processing
Action: "Sends a prompt to Copilot"

Purpose: Passes the classification task to your custom-trained Copilot Agent.

Configuration:

Use the Copilot Agent you created earlier.

Pass the text from GPT step to Copilot for final classification decision.

🔹 Final Metadata Update (HTTP to SharePoint)
Step 11: Send an HTTP request to SharePoint
Action: "Send an HTTP request to SharePoint"

Purpose: Update SharePoint metadata with the determined sensitivity label.

Configuration:

Site Address: Your SharePoint URL

Method: POST

URI:

bash
Copy
Edit
_api/web/GetFileByServerRelativeUrl('[FilePathVariable]')/ListItemAllFields
Headers:

json
Copy
Edit
{
  "Accept": "application/json;odata=verbose",
  "Content-Type": "application/json;odata=verbose",
  "IF-MATCH": "*",
  "X-HTTP-Method": "MERGE"
}
Body:

json
Copy
Edit
{
  "__metadata": {
    "type": "SP.Data.Shared_x0020_DocumentsItem"
  },
  "AISensitivity": "[classification_label_from_copilot]"
}
⚠ Important: Preventing Double-Triggers
To avoid continuous re-triggering:

Add a condition after the initial trigger action:

Condition expression:

scss
Copy
Edit
empty(triggerOutputs()?['body/AISensitivity'])
If true (AISensitivity is empty): Continue flow.

If false: Terminate flow immediately to prevent redundant processing.

🎉 Final Result: Your Power Automate flow is now fully configured and will autonomously:

Detect document creation/modification.

Extract text content reliably (OCR or Graph API as alternative).

Classify documents accurately using GPT & Copilot Studio.

Update metadata in SharePoint with sensitivity labels automatically.

Avoid redundant execution due to double-trigger prevention.

This autonomous process ensures documents within your organization are consistently and correctly classified, significantly reducing manual effort and improving compliance.










=======================================================================================
OLDER INSTRUCTIONS FOR END USER AGENT - NOT FOR AUTONOMOUS
=======================================================================================


Custom Instructions for Classification Agent


Name: Classification Agent 

Description:

This agent will reference and interpret the Document Classification Scheme for Medium-to-Large Organisations as the central knowledge base, accurately assign the correct label, and then offer a concise justification for its decision

Knowledge Source:  Document Classification Scheme for Medium-to-Large Organizat.docx

Custom Instructions:

- **Role and Purpose**

- You (“Classification Agent”) act as an expert that determines a document’s proper sensitivity label based on the guidelines from the document titled:  
***“Document Classification Scheme for Medium-to-Large Organizations”*** (hereafter referred to as the “Knowledge Source”).
- You must **read** or **review** the user-provided content, then compare it to the **Knowledge Source** definitions.
- **Classification Labels**

- Consult the five classification tiers and sub-labels described in the **Knowledge Source**:
    1. **Personal**
    2. **Public**
    3. **General**
        - General \ Anyone (unrestricted)
        - General \ All Employees (unrestricted)
    4. **Confidential**
        - Confidential \ Anyone (unrestricted)
        - Confidential \ All Employees
        - Confidential \ Trusted People
    5. **Highly Confidential**
        - Highly Confidential \ All Employees
        - Highly Confidential \ Specific People
- **Decision Criteria**

- Use the detailed definitions, examples, and usage guidelines from the **Knowledge Source** to match the document’s content with the most suitable classification.
- Key points to consider:
    - **Sensitivity Level**: Does it contain personal data, regulated info, trade secrets, or routine internal details?
    - **Intended Audience**: Is it meant for the public, internal staff, specific individuals, or no one else?
    - **Impact of Exposure**: Could disclosure cause legal, financial, or reputational harm?
- **Output Format**

- Your response must include **two elements only**:
    1. **Classification Label**: One of the valid labels (e.g., “General \ All Employees”).
    2. **Brief Explanation**: A concise 1–2 sentence rationale indicating **why** that label applies, without revealing detailed internal decision rules.
- **Restrictions and Scope**

- **Do Not** quote extensive text from the Knowledge Source or provide a multi-paragraph explanation.
- **Do Not** reference or share your internal reasoning steps or mention instructions like “As per Step 3 in the knowledge source.”
- **Do Not** provide speculation or hypothetical scenarios beyond what the user’s text suggests.
- **Do** keep your explanation strictly relevant to the immediate content context (1–2 sentences).
- **Prohibited Content**

- **No** detailed references to internal frameworks (e.g., ISO 27001, NIST SP 800-53) in your final answer.
- **No** step-by-step methodology or debug logs.
- **No** policy text quoting from the Knowledge Source. You may only reference it implicitly.
- **Example Response**

- **Input**: A user shares a draft of a public press release that’s clearly intended for distribution to external news outlets.
- **Output**:
    1. **Classification Label**: Public
    2. **Brief Explanation**: “This content is officially approved for external audiences and poses no confidentiality risks.”
- **Final Reminder**

- Always select **one** classification label that best fits the content.
- Provide only a short explanation.
- Rely on the **Knowledge Source** whenever in doubt about definitions or sub-label usage.



======================================================================================

EXAMPLE CLASSIFICATION DOCUMENT POLICY:

Document Classification Scheme for Medium-to-Large Organisations 

Executive Summary 

A robust document classification scheme is essential for protecting sensitive information, ensuring regulatory compliance, and mitigating risks in today's data-driven environment. This five-tier framework—Personal, Public, General, Confidential, and Highly Confidential—provides clear definitions, real-world examples, and actionable guidelines for handling, storing, and sharing documents. By following best practices (e.g., ISO 27001, NIST SP 800-53) and tailoring these categories to their specific organizational structure, companies can achieve an optimal balance between security, productivity, and compliance. 

 

1. Introduction 

Purpose of Document Classification 

Document classification is a foundational element of information governance. By categorizing documents based on sensitivity, criticality, and regulatory requirements, organizations can: 

Protect valuable information proportionally to its risk. 

Comply with legal and industry mandates (e.g., GDPR, HIPAA, PCI-DSS). 

Streamline access so that employees and partners get the information they need—without exposing sensitive data. 

Mitigate risks by ensuring the proper controls are in place for each classification level. 

Importance of a Structured Classification Scheme 

An effective classification scheme: 

Prevents unauthorized access to sensitive information. 

Supports regulatory compliance (GDPR, HIPAA, PCI-DSS) and avoids fines or reputational damage. 

Increases operational efficiency by providing clarity on how documents should be handled and shared. 

Reduces over- or under-classification, each of which can create bottlenecks or security gaps, respectively. 

Scope of This Framework 

This document defines five classification tiers—Personal, Public, General, Confidential, and Highly Confidential—and their sub-labels where applicable. It offers examples, handling guidelines, and procedures for implementation, monitoring, and reclassification. Although designed for medium-to-large organizations, these classifications can be adapted for different sectors and sizes. 

 

2. Classification Levels 

Below is the complete classification structure with sub-labels. Each section includes definitions, examples, justifications, handling guidelines, and risks of misclassification. 

 

2.1 Personal 

Definition: 

Non-business data intended for personal use only. 

Typically created and managed by individuals, with no organizational oversight required. 

Examples: 

Personal notes (e.g., a user’s private notepad). 

Employee’s vacation photos stored on a personal drive. 

Drafts of personal memos unrelated to business operations. 

Justification for Classification: 

Ensures the organization’s information governance focuses on corporate data rather than private content. 

Respects employee privacy by acknowledging that certain files are strictly personal. 

Handling and Sharing Guidelines: 

Label documents as “Personal” (where labeling systems allow). 

Store in a user’s personal drive, not intended for broad internal distribution. 

Do Not Share with other employees unless the user deems it necessary. 

Risks of Misclassification: 

Storing genuinely personal data under corporate classifications may trigger unnecessary audits or data retention policies. 

Conversely, mixing personal and business data in the same storage location can lead to data protection issues. 

 

2.2 Public 

Definition: 

Business data explicitly approved for external (public) consumption. 

Disclosure poses little to no harm to the organization. 

Examples: 

Official press releases, finalized marketing brochures or campaigns, publicly posted job ads. 

Content explicitly designated “for media” or “for public website.” 

Any material cleared by legal/PR teams for external distribution. 

Justification for Classification: 

Allows the organization to clearly identify information safe to share publicly. 

Ensures that only authorized data is communicated to outside parties. 

Handling and Sharing Guidelines: 

Label such documents as “Public.” 

Store in areas accessible to all employees, and potentially external distribution platforms (e.g., a public website). 

No Special Controls typically necessary, as the information is intended for broad release. 

Risks of Misclassification: 

Over-classification can impede marketing or PR efforts. 

Under-classification could inadvertently disclose sensitive information that was not intended for public view. 

 

2.3 General 

Business data that is not intended for broad public consumption but may not be highly sensitive. General is subdivided into: 

General \ Anyone (unrestricted) 

Definition: Business data that can be shared internally and potentially with external partners if appropriate. 

Examples: Basic project updates, non-sensitive operational checklists, informational handbooks that contain no confidential data. 

Usage:  

Typically accessible to all employees and can be shared externally when necessary (e.g., with consultants or vendors), provided data owners approve. 

Handling and Sharing:  

Label as “General \ Anyone.” 

Store in shared drives with open access for employees. 

For external sharing, confirm it’s acceptable to do so at this classification. 

General \ All Employees (unrestricted) 

Definition: Business data intended only for internal audiences across the entire organization. 

Examples: Internal organizational charts, departmental announcements, non-sensitive internal reports. 

Usage:  

Accessible to any employee internally. 

Not intended for external parties; must be re-labeled (e.g., “General \ Anyone”) if it needs to be shared outside. 

Handling and Sharing:  

Label as “General \ All Employees.” 

Store in platforms restricted to employees only. 

Avoid accidental external distribution. 

Risks of Misclassification (General Overall): 

Over-classifying low-sensitivity items as “Confidential” or “Highly Confidential” can slow collaboration. 

Under-classifying more sensitive items as “General” can lead to data leaks or other issues. 

 

2.4 Confidential 

Indicates sensitive business data that could cause damage if misused or improperly shared. Confidential sub-labels dictate who can view and how it’s protected. 

Confidential \ Anyone (unrestricted) 

Definition: Still sensitive, but encryption or extremely limited permissions are not strictly required. 

Examples: Draft client proposals without final numbers, internal memos discussing potential business initiatives, lesser financial details. 

Usage:  

Caution is needed when sharing externally. 

Typically used when the organization acknowledges the sensitivity but does not require fully restricted distribution. 

Confidential \ All Employees 

Definition: Sensitive info accessible by the entire employee base, but not external parties without reclassification or approval. 

Examples: Internal performance dashboards, monthly/quarterly financial summaries (not yet public), certain managerial guidelines. 

Usage:  

Encryption may be required during transmission. 

All employees can access, but external sharing is tightly controlled. 

Confidential \ Trusted People 

Definition: Data restricted to specifically designated groups or individuals (who may need to share among themselves or with closely vetted external parties). 

Examples: Client contracts, statements of work, pre-sales RFPs that reference sensitive client data, MOU drafts. 

Usage:  

Sharing outside the designated group requires explicit permission. 

Often subject to NDAs or legal constraints. 

Risks of Misclassification (Confidential Overall): 

Under-classifying confidential information can expose the organization to reputational damage, contractual penalties, or regulatory fines. 

Over-classifying routine internal documents as “Confidential” can reduce productivity and create unnecessary administrative overhead. 

 

2.5 Highly Confidential 

Top-tier sensitivity level reserved for extremely critical or regulated data, where disclosure could bring severe legal, financial, or operational harm. 

Highly Confidential \ All Employees 

Definition: The organization deems it necessary for everyone to see or access, but it remains extremely sensitive (e.g., major strategic announcements or reorganization plans). 

Examples: Internal memos about significant company restructuring or layoffs, organization-wide legal notices. 

Usage:  

Strict monitoring—access logs and audits are often required. 

Encryption both at rest and in transit is strongly advised. 

Highly Confidential \ Specific People 

Definition: The strictest classification, limited to named individuals or roles. 

Examples: Merger & Acquisition (M&A) documentation, advanced R&D project files (e.g., proprietary source code), highly regulated personal data (e.g., protected health information). 

Usage:  

Access is granted on a need-to-know basis only. 

Heavy encryption, multi-factor authentication, and frequent auditing are best practices. 

Risks of Misclassification (Highly Confidential Overall): 

Under-classifying can lead to catastrophic data breaches, intense legal or regulatory consequences, and loss of competitive advantage. 

Over-classifying less critical data at this level can impede essential workflows and hamper timely decisions. 

 

3. Implementation Guidelines 

3.1 Labeling and Marking Documents 

Personal: Mark clearly as “Personal” where labeling tools allow. 

Public: Mark as “Public” in file metadata or footers if feasible. 

General: Indicate “General \ Anyone” or “General \ All Employees” in file headers/footers or metadata fields. 

Confidential: Label as “Confidential \ [Sub-label]” (e.g., “Confidential \ Trusted People”). 

Highly Confidential: Label as “Highly Confidential \ [Sub-label]” (e.g., “Highly Confidential \ Specific People”). 

3.2 Access Controls 

Role-Based Access Control (RBAC): Assign roles and groups consistent with classification levels. 

Multi-Factor Authentication (MFA): Enforce MFA for Confidential and Highly Confidential data. 

Auditing and Logging: Regularly review who accessed which files, especially in the Confidential and Highly Confidential tiers. 

3.3 Storage and Transmission 

Personal: Typically stored in user-specific storage (personal drives). 

Public: Can be on public-facing platforms (websites, externally accessible repositories). 

General: In shared drives/intranets with or without external access, depending on the sub-label. 

Confidential: Use secure, access-controlled file systems; consider encryption especially for external sharing. 

Highly Confidential: Always store in encrypted systems; strictly limit user privileges. 

3.4 Training and Awareness 

Conduct classification and handling training for new hires and provide annual refresher sessions. 

Make classification guidelines easily accessible (e.g., an internal wiki). 

Offer clear escalation channels for questions or suspected misclassification events. 

3.5 Reclassification Procedures 

Regularly review document classifications (e.g., data that was Highly Confidential may move to Confidential after certain events). 

Document rationale for all reclassifications and secure approval from data owners or compliance teams. 

Update labels, storage locations, and access controls accordingly. 

 

4. Real-World Scenarios 

Scenario 1: M&A Documentation 

Context: A company is preparing for an acquisition. Early M&A documents often include trade secrets and major strategic details. 

Classification: “Highly Confidential \ Specific People” until public announcements are made. 

Implementation:  

Store in secure, encrypted repositories. 

Only senior executives, legal counsel, and select advisors have access. 

Reclassification:  

After the acquisition is announced, parts of this documentation may move to “Confidential \ Trusted People” or even “Public” (e.g., press releases). 

Scenario 2: Employee HR Records 

Context: Performance reviews, medical leave requests, payroll info, etc. 

Classification: Typically “Confidential \ Trusted People,” shared only with HR and the employee’s manager. 

Implementation:  

Use specialized HR software with restricted role-based access. 

Encrypt files at rest and in transit if they contain personal identifiers. 

Reclassification:  

Aggregated, anonymized data for organizational reporting can be labeled “General \ All Employees” or even “Public” if it’s part of an external communication (e.g., broad market data with no personal info). 

Scenario 3: Product Launch 

Context: Unreleased product designs or marketing strategies. 

Classification: “Highly Confidential \ Specific People” for R&D or design teams during early stages. 

Implementation:  

Secure storage in design systems with limited access. 

Enforce encryption when sharing prototypes or design documentation. 

Reclassification:  

After launch, design documents may remain “Confidential \ All Employees” for broader internal reference, while externally shared marketing materials become “Public.” 

Scenario 4: General Company Announcements 

Context: An internal memo about a new cafeteria policy or monthly newsletter. 

Classification: Usually “General \ All Employees.” 

Implementation:  

Send via company intranet or broadcast email. 

No special encryption required. 

Reclassification:  

If it’s later turned into an external blog post or press mention, reclassify to “Public.” 

 

5. Conclusion 

A well-defined document classification scheme is the cornerstone of robust information governance. By adopting the Personal, Public, General, Confidential, and Highly Confidential categories (and their sub-labels), organizations can tailor access controls, enhance collaboration, and comply with legal obligations while protecting vital assets. Successful implementation relies on ongoing training, consistent labeling, and regular review of how documents are classified as business needs evolve. Through a combination of structured processes, technology, and organizational awareness, this classification framework will mitigate risks and foster a culture of security and accountability. 

 
