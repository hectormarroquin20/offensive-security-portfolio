# Software Security Re-Architecture Case Study: Mitigating SQL Injection & Modernizing Legacy Image Handling ([Obfuscated Financial Portal])

## 📌 Executive Summary
This case study documents a comprehensive security re-architecture initiative executed for a high-traffic financial consultation portal. Following a rigorous security audit that uncovered critical SQL Injection (SQLi) vulnerabilities—specifically embedded within the application's legacy binary image retrieval mechanism—the engineering and security teams advised against iterative patching. Instead, leadership authorized a complete greenfield rewrite. The legacy monolith was successfully replaced with a decoupled, modern API-driven architecture utilizing **Hibernate ORM** for secure data handling and an innovative **Base64/64-bit stream rendering** pattern to deliver dynamic visual assets directly to the browser without exposing physical file paths or vulnerable download routes.

## 🛠️ Technical Scope & Legacy Vulnerability Analysis
* **Core Application:** Financial Client Consultation and Document Retrieval Portal.
* **Legacy Vulnerability:** Critical SQL Injection via image database queries (unsafe concatenation and direct BLOB handling through vulnerable query parameters).
* **Core Tech Stack (Modernized):** Java / Spring Boot, Hibernate ORM, RESTful APIs, Frontend Framework, Base64 Stream Encoding.
* **Methodology:** Secure Code Review, Threat Modeling, Architecture Refactoring, and Secure-by-Design Principles implementation.

---

## 🔍 The Security Breakdown: Why a Rewrite Outweighed Patching

### Phase 1: The Security Audit & SQL Injection Discovery
* During a routine security assessment of the consultation portal, multiple critical SQL Injection (CVE-class) vectors were discovered.
* **The Root Cause:** The legacy application stored binary image data directly inside database tables. When users requested documents or profile images, the application dynamically queried the database using string-concatenated SQL queries, failing to properly sanitize input parameters or isolate binary data streams.
* **The Risk:** Attackers could manipulate query parameters to execute arbitrary database commands, potentially extracting sensitive financial records, system tables, or dumping full user datasets.

### Phase 2: Strategic Decision — Rewrite vs. Patch
* Iterative patching (adding input filters or regex validation) was evaluated but rejected due to deep architectural flaws:
  * The tightly coupled monolith made secure data abstraction extremely difficult.
  * Direct file/blob database handling bloated storage and introduced severe performance bottlenecks.
  * Exposed file download endpoints presented persistent path traversal and insecure direct object reference (IDOR) risks.
* **Strategic Pivot:** Leadership authorized a complete architectural overhaul to decouple the system, introduce robust object-relational mapping, and eliminate physical file paths entirely.

---

## 🏗️ Modern Architecture Redesign & Secure Engineering

### Phase 1: Decoupled API-Driven Architecture
* The legacy monolithic structure was dismantled and refactored into a clean, decoupled architecture featuring a secure RESTful backend API and a modern responsive frontend.
* Communication was strictly standardized via JSON payloads over HTTPS, removing legacy direct database-to-UI rendering pipelines.

### Phase 2: Secure Data Layer with Hibernate ORM
* Data management was migrated to **Hibernate ORM**, completely eliminating raw, concatenated SQL queries.
* Hibernate's built-in parameterized queries and HQL/Criteria APIs automatically enforce type safety and protect against SQL Injection by default, abstracting database operations into clean domain objects.

### Phase 3: Base64 / 64-Bit Inline Stream Rendering
* To eliminate the security risks associated with physical file download paths, IDs, or static directory traversal, the image and document handling pipeline was completely reimagined:
  * Instead of storing file paths or generating temporary download links, binary assets are converted into optimized **64-bit (Base64) encoded strings** at the backend service layer.
  * These Base64 strings are transmitted securely inside JSON API responses.
  * The frontend receives the string and instructs the browser to render the asset directly via data URIs (e.g., `data:image/png;base64,...`), completely removing the need for physical file storage references, download routes, or static asset endpoints.

---

### 💻 Technical Implementation Highlights

```java
// Conceptual Backend Example: Secure Entity Management & Base64 Transformation via Hibernate
@Entity
@Table(name = "client_documents")
public class ClientDocument {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "doc_name")
    private String docName;

    @Lob
    @Column(name = "binary_data")
    private byte[] binaryData; // Managed securely via Hibernate ORM (Parameterized execution)

    // Method to convert binary stream into a secure 64-bit string for browser inline rendering
    public String getBase64EncodedStream() {
        if (this.binaryData != null) {
            return Base64.getEncoder().encodeToString(this.binaryData);
        }
        return "";
    }
}
```

```JSON
// Modern API Response Payload Example
{
  "status": "success",
  "documentId": 1042,
  "title": "Account_Verification",
  "mimeType": "image/png",
  "renderData": "iVBORw0KGgoAAAANSUhEUgAA..." // 64-bit Base64 string rendered directly by the browser
}
```