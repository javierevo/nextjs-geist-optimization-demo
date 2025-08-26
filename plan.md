```markdown
# Detailed Implementation Plan for MasterClass Certificate Generation

This plan outlines the steps to implement a certificate generation system for the MasterClass. The system will include a secure form for participants to enter their email and access key, an API endpoint to validate the credentials and generate a PDF certificate, and modern UI components that follow best practices. All error cases are handled gracefully, and the solution uses local JSON storage for participant data along with PDF generation using a Node.js PDF library.

---

## 1. New Data File for Participant Records

**File:** `src/data/participants.json`  
**Purpose:** Store an array of participant objects with their registration email, full name, and shared access key.  
**Content Example:**
```json
[
  {
    "email": "example@example.com",
    "name": "Juan Pérez",
    "accessKey": "ABC123"
  }
]
```
> **Note:** This file is used by the API endpoint for credential validation. Ensure its path is correct and that error handling is added in case the file is missing or unreadable.

---

## 2. Dependency Update in Package Configuration

**File:** `package.json`  
**Action:**  
- Add a dependency for PDF generation (e.g., using [pdfkit](https://github.com/foliojs/pdfkit)).  
- Run `npm install pdfkit` after adding the dependency.  
**Example Addition:**
```json
"dependencies": {
  "pdfkit": "^0.12.3",
  // ... other dependencies
}
```
> **Best Practice:** Ensure that the package-lock.json file is updated and commit both files to version control.

---

## 3. UI Implementation: Certificate Request Form

**File:** `src/app/certificate/page.tsx`  
**Responsibilities:**
- Render a modern, clean form with inputs for the email and access key.
- Use proper labels, placeholder text in Spanish, and error message display.
- Include a submit button and a loading spinner for asynchronous operations.

**Implementation Outline:**
- **Layout & Styling:**  
  - Use a centered card layout with ample spacing and clear typography.
  - Apply styles via global CSS (in `src/app/globals.css`) or inline CSS modules.
- **Form Elements:**  
  - Input for "Correo electrónico" with validations.
  - Input for "Clave compartida" with validations.
  - A submit button styled for clarity.
- **Form Submission:**  
  - On submit, call a client-side function that sends a POST request to `/api/generate-certificate` with email and access key in JSON.
  - Handle response errors by showing a clear error message in red below the form.
  - On success, receive the PDF as a Blob and trigger a file download using URL.createObjectURL.

**Sample Code Snippet (Simplified):**
```tsx
"use client";
import { useState } from "react";

export default function CertificateForm() {
  const [email, setEmail] = useState("");
  const [accessKey, setAccessKey] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError("");
    if (!email || !accessKey) {
      setError("Ambos campos son requeridos.");
      return;
    }
    setLoading(true);
    try {
      const res = await fetch("/api/generate-certificate", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, accessKey }),
      });
      if (!res.ok) {
        const json = await res.json();
        throw new Error(json.message || "Error en la validación.");
      }
      const blob = await res.blob();
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "Certificado_MasterClass.pdf";
      document.body.appendChild(a);
      a.click();
      a.remove();
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ maxWidth: "500px", margin: "3rem auto", padding: "2rem", border: "1px solid #e5e5e5", borderRadius: "8px" }}>
      <h1 style={{ textAlign: "center", marginBottom: "1rem" }}>Genera tu Certificado</h1>
      <p style={{ textAlign: "center", marginBottom: "2rem", color: "#555" }}>
        Ingresa tu correo registrado y la clave compartida.
      </p>
      <form onSubmit={handleSubmit}>
        <label style={{ display: "block", marginBottom: "0.5rem" }}>Correo electrónico</label>
        <input
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          placeholder="ejemplo@correo.com"
          style={{ width: "100%", padding: "0.75rem", marginBottom: "1rem", borderRadius: "4px", border: "1px solid #ccc" }}
          required
        />
        <label style={{ display: "block", marginBottom: "0.5rem" }}>Clave compartida</label>
        <input
          type="text"
          value={accessKey}
          onChange={e => setAccessKey(e.target.value)}
          placeholder="Ingresa la clave"
          style={{ width: "100%", padding: "0.75rem", marginBottom: "1rem", borderRadius: "4px", border: "1px solid #ccc" }}
          required
        />
        {error && <div style={{ color: "red", marginBottom: "1rem" }}>{error}</div>}
        <button
          type="submit"
          style={{ width: "100%", padding: "0.75rem", backgroundColor: "#0070f3", color: "#fff", border: "none", borderRadius: "4px" }}
          disabled={loading}
        >
          {loading ? "Generando..." : "Generar Certificado"}
        </button>
      </form>
    </div>
  );
}
```
> **UI Note:** No external image services or icon libraries are used. The design employs typography, spacing, and color for a modern look.

---

## 4. API Endpoint for Certificate Generation

**File:** `src/app/api/generate-certificate/route.ts`  
**Responsibilities:**
- Accept POST requests with JSON payload (email and access key).
- Validate the presence and correctness of the input by:
  - Reading the `src/data/participants.json` file.
  - Matching the email and access key values with an existing record.
- Generate a PDF certificate using `pdfkit` if validation passes.
- Return the PDF with proper headers (Content-Type and Content-Disposition for file download).
- Handle errors (missing fields, file read errors, validation failure) with appropriate HTTP status codes.

**Implementation Outline:**
- **Input Validation:** Verify that both email and key are provided.
- **Data Access:** Use the Node.js `fs` module to read `participants.json`. Use try-catch to handle file I/O errors.
- **Credential Validation:** Search for a participant object where both email and accessKey match.
- **PDF Generation Using pdfkit:** 
  - Create a new PDFDocument.
  - Set the document metadata.
  - Add certificate title, participant’s name (as stored), details of 3 academic hours, and any additional decorative text.
- **Response:** Stream the generated PDF back to the client.
- **Error Handling:** Return JSON error messages (e.g., 400 for missing fields, 401 for invalid credentials, 500 for internal errors).

**Sample Code Outline:**
```ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import fs from "fs/promises";
import path from "path";
import PDFDocument from "pdfkit";

export async function POST(request: NextRequest) {
  try {
    const { email, accessKey } = await request.json();
    if (!email || !accessKey) {
      return NextResponse.json({ message: "Ambos campos son requeridos." }, { status: 400 });
    }

    // Load participant data
    const dataPath = path.join(process.cwd(), "src", "data", "participants.json");
    let participantsData;
    try {
      const fileContents = await fs.readFile(dataPath, "utf8");
      participantsData = JSON.parse(fileContents);
    } catch (err) {
      return NextResponse.json({ message: "Error interno al leer los datos de participantes." }, { status: 500 });
    }

    // Validate participant credentials
    const participant = participantsData.find((p: any) => p.email === email && p.accessKey === accessKey);
    if (!participant) {
      return NextResponse.json({ message: "Credenciales inválidas." }, { status: 401 });
    }

    // Generate PDF certificate using pdfkit
    const doc = new PDFDocument({ size: "A4", margin: 50 });
    let buffers: any[] = [];
    doc.on("data", buffers.push.bind(buffers));
    doc.on("end", () => {});

    // Add certificate design
    doc.fontSize(24).text("Certificado de Participación", { align: "center" });
    doc.moveDown(1);
    doc.fontSize(16).text(`Este certificado acredita la participación de:`, { align: "center" });
    doc.moveDown(0.5);
    doc.fontSize(20).text(participant.name, { align: "center", underline: true });
    doc.moveDown(1);
    doc.fontSize(14).text("En la MásterClass, con una duración de 3 horas académicas.", { align: "center" });
    doc.moveDown(2);
    doc.fontSize(12).text("Gracias por participar y esperamos su continuidad en futuros eventos.", {
      align: "center",
    });
    doc.end();

    return new NextResponse(Buffer.concat(buffers), {
      status: 200,
      headers: {
        "Content-Type": "application/pdf",
        "Content-Disposition": 'attachment; filename="Certificado_MasterClass.pdf"',
      },
    });
  } catch (error: any) {
    return NextResponse.json({ message: "Error interno del servidor: " + error.message }, { status: 500 });
  }
}
```
> **Important:** Ensure that the relative path to `participants.json` is correct and that proper error messages are given if file reading or PDF generation fails.

---

## 5. Testing and Verification

- **Local Testing via Curl:**  
  Use the following command to test the API endpoint:
  ```bash
  curl -X POST http://localhost:3000/api/generate-certificate \
    -H "Content-Type: application/json" \
    -d '{"email": "example@example.com", "accessKey": "ABC123"}' \
    -o output.pdf \
    --silent --show-error
  ```
  Validate that the PDF is generated and that invalid inputs produce proper error responses.

- **Client-Side Testing:**  
  Verify in the browser that when the form is submitted:
  - Invalid inputs show error messages.
  - Valid inputs trigger a download of the certificate PDF.
  - The UI remains responsive with appropriate loading states.

---

## Summary

- Created a new data file (`src/data/participants.json`) for storing participant information.
- Updated `package.json` to include the `pdfkit` dependency for PDF generation.
- Developed a modern, responsive form UI in `src/app/certificate/page.tsx` with validations and error handling.
- Implemented an API endpoint in `src/app/api/generate-certificate/route.ts` to validate inputs, generate a PDF certificate, and return it with correct headers.
- Ensured error handling for missing fields, invalid credentials, and file I/O issues.
- Provided detailed instructions for local testing via curl and browser-based validations.
