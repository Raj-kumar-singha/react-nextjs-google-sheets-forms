# react-nextjs-google-sheets-forms
Send Next.js or React.js form submissions (Formik + Yup) straight into Google Sheets via a Google Apps Script Web App — no backend required.

Git Repo Link : - [Github](https://github.com/Raj-kumar-singha/ReactJs-Multiple-Functionalites)
## Prerequisites
- A Google account
- A Google Sheet you can edit
- A React or Next.js app (Tailwind optional)
- Node 16+ (for your app)

## 1) Create your Google Sheet
   
  1. Create (or open) a Google Sheet.
  2. Add a tab named ContactUs with these headers in row 1:
```Timestamp | Name | Email | Phone | Message | Source | Status```
  3. Copy your Sheet ID from the URL:
```https://docs.google.com/spreadsheets/d/<SHEET_ID>/edit#gid=0```

## 2) Add the Apps Script (writes rows into the sheet)
1. In your Sheet, go to Extensions → Apps Script.
2. Replace the default file’s contents with the code below.
3. Update SHEET_ID to your actual ID
```javascript
// ====== CONFIG ======
const SHEET_ID   = 'PASTE_YOUR_SHEET_ID_HERE';
const SHEET_NAME = 'ContactUs';

function getSheet_() {
  const ss = SpreadsheetApp.openById(SHEET_ID);
  let sh = ss.getSheetByName(SHEET_NAME);
  if (!sh) sh = ss.insertSheet(SHEET_NAME);
  const headers = ['Timestamp', 'Name', 'Email', 'Phone', 'Message', 'Source', 'Status'];
  if (sh.getLastRow() === 0) {
    sh.appendRow(headers);
  } else {
    const existing = sh.getRange(1, 1, 1, headers.length).getValues()[0];
    if (existing.join('¦') !== headers.join('¦')) {
      sh.insertRows(1);
      sh.getRange(1, 1, 1, headers.length).setValues([headers]);
    }
  }
  // Force the Phone column (D) to TEXT so "+971..." isn't parsed as a formula
  sh.getRange('D:D').setNumberFormat('@');
  return sh;
}

function doGet() { return json_({ ok: true }); }

function doPost(e) {
  try {
    if (!e || !e.postData) return json_({ success:false, message:'No payload' }, 400);

    const ct  = (e.postData.type || '').toLowerCase();
    const raw = e.postData.contents || '';
    const body = ct.includes('application/json') ? JSON.parse(raw) : qsToObj_(raw);

    const name    = String(body.name || '').trim();
    const email   = String(body.email || '').trim().toLowerCase();
    const phoneIn = String(body.phone || '').trim();
    const message = String(body.message || '').trim();
    const source  = String(body.source || 'contact-form').trim();

    // Basic validation
    if (name.length < 2)                          return json_({ success:false, message:'Name is required' }, 400);
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) return json_({ success:false, message:'Invalid email' }, 400);
    if (!/^\+?[0-9()\-\s]{7,15}$/.test(phoneIn))   return json_({ success:false, message:'Invalid phone' }, 400);
    if (message.length < 10)                      return json_({ success:false, message:'Message too short' }, 400);

    // If value starts with "+", prefix an apostrophe to force text
    const phoneForSheet = phoneIn.startsWith('+') ? "'" + phoneIn : phoneIn;

    const sh = getSheet_();
    const nextRow = sh.getLastRow() + 1;
    sh.getRange(nextRow, 1, 1, 7).setValues([
      [new Date(), name, email, phoneForSheet, message, source, 'received']
    ]);

    return json_({ success:true, message:'Thanks! We’ll get back to you shortly.' }, 200);
  } catch (err) {
    return json_({ success:false, message:String(err) }, 500);
  }
}

// ---------- helpers ----------
function qsToObj_(qs) {
  return (qs || '').split('&').reduce((acc, pair) => {
    if (!pair) return acc;
    const [k, v=''] = pair.split('=');
    acc[decodeURIComponent(k)] = decodeURIComponent(v.replace(/\+/g,' '));
    return acc;
  }, {});
}

function json_(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

```
## 3) Deploy the Apps Script as a Web App
1. Deploy → New deployment
2. Select type: Web app
3. Execute as: Me
4. Who has access: Anyone
5. Deploy, then copy the Web app URL (ends with /exec)

## 4) Add the React/Next.js Contact form
 - "Works in React or Next.js. Uses application/x-www-form-urlencoded to avoid CORS preflight. No mode: "no-cors"."
 - components/SubmitForm.tsx
```javascript
import React from 'react';
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as Yup from 'yup';
import { toast } from 'react-hot-toast';

interface ContactFormValues {
  name: string;
  email: string;
  phone: string;
  message: string;
}

// Paste your /exec URL here
const CONTACT_APPS_SCRIPT_URL = 'https://script.google.com/macros/s/PASTE_EXEC/exec';

const schema = Yup.object({
  name: Yup.string().trim().min(2, 'Enter your full name').required('Name is required'),
  email: Yup.string().trim().email('Enter a valid email').required('Email is required'),
  phone: Yup.string().trim().matches(/^\+?[0-9()\-\s]{7,15}$/, 'Enter a valid phone number').required('Phone is required'),
  message: Yup.string().trim().min(10, 'Message must be at least 10 characters').required('Message is required'),
});

const SubmitForm: React.FC = () => {
  const initialValues: ContactFormValues = { name: '', email: '', phone: '', message: '' };

  const handleSubmit = async (
    values: ContactFormValues,
    { resetForm, setSubmitting }: { resetForm: () => void; setSubmitting: (s: boolean) => void }
  ) => {
    try {
      const res = await fetch(CONTACT_APPS_SCRIPT_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8' },
        body: new URLSearchParams({
          name: values.name.trim(),
          email: values.email.trim(),
          phone: values.phone.trim(),
          message: values.message.trim(),
          source: 'contact-form',
        }),
      });

      const data = await res.json(); // Apps Script responds with JSON
      if (!res.ok || data.success === false) throw new Error(data.message || 'Failed to submit form');

      toast.success(data.message || 'Submitted successfully!');
      resetForm();
    } catch (err: any) {
      toast.error(err?.message || 'Failed to submit form');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 flex flex-col items-center justify-center p-8">
      <div className="bg-white rounded-lg shadow-lg p-8 max-w-md w-full">
        <h1 className="text-2xl font-bold text-gray-800 mb-2">Contact Us</h1>
        <p className="text-gray-600 mb-6">Fill the form and we’ll contact you soon.</p>

        <Formik initialValues={initialValues} validationSchema={schema} onSubmit={handleSubmit}>
          {({ isSubmitting }) => (
            <Form className="space-y-4">
              <div>
                <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-1">Name</label>
                <Field id="name" name="name" type="text" placeholder="Your full name"
                  className="w-full px-4 py-3 border rounded-lg text-gray-700 focus:outline-none focus:ring-2 focus:ring-green-700 border-gray-300"
                  disabled={isSubmitting} />
                <ErrorMessage name="name" component="div" className="text-red-500 text-sm mt-1" />
              </div>

              <div>
                <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-1">Email</label>
                <Field id="email" name="email" type="email" placeholder="you@example.com"
                  className="w-full px-4 py-3 border rounded-lg text-gray-700 focus:outline-none focus:ring-2 focus:ring-green-700 border-gray-300"
                  disabled={isSubmitting} />
                <ErrorMessage name="email" component="div" className="text-red-500 text-sm mt-1" />
              </div>

              <div>
                <label htmlFor="phone" className="block text-sm font-medium text-gray-700 mb-1">Phone</label>
                <Field id="phone" name="phone" type="tel" placeholder="+91 98765 43210"
                  className="w-full px-4 py-3 border rounded-lg text-gray-700 focus:outline-none focus:ring-2 focus:ring-green-700 border-gray-300"
                  disabled={isSubmitting} />
                <ErrorMessage name="phone" component="div" className="text-red-500 text-sm mt-1" />
              </div>

              <div>
                <label htmlFor="message" className="block text-sm font-medium text-gray-700 mb-1">Message</label>
                <Field as="textarea" id="message" name="message" rows={5} placeholder="How can we help?"
                  className="w-full px-4 py-3 border rounded-lg text-gray-700 focus:outline-none focus:ring-2 focus:ring-green-700 border-gray-300"
                  disabled={isSubmitting} />
                <ErrorMessage name="message" component="div" className="text-red-500 text-sm mt-1" />
              </div>

              <button type="submit" disabled={isSubmitting}
                className="w-full py-3 rounded-sm text-green-700 text-[16px] bg-transparent border border-green-700 font-bold hover:bg-green-700 hover:text-white transition-colors disabled:opacity-50 disabled:cursor-not-allowed">
                {isSubmitting ? 'Submitting…' : 'Submit'}
              </button>
            </Form>
          )}
        </Formik>
      </div>
    </div>
  );
};

export default SubmitForm;

```




## Take a Pull from my repo and explore more
- step-by-step guide
```bash
git pull https://github.com/Raj-kumar-singha/ReactJs-Multiple-Functionalites.git

npm i

npm run dev

Check the "/submit-form" routes

check the code of "SubmitForm" component for better understanding
```

## Output in spreadsheet
<img width="1823" height="1023" alt="image" src="https://github.com/user-attachments/assets/704d82c9-b263-4207-b1af-e51a60d88a3b" />

## 🌐 Connect with Me
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/raj-kumar-singha-63a7b5169)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Raj-kumar-singha)

**Still have questions or want to connect?**
- 💼 Follow me on [LinkedIn](https://www.linkedin.com/in/raj-kumar-singha-63a7b5169/)
- 💬 Send me a DM on LinkedIn for collaborations
- ⭐ Star this repo if you found it helpful!
