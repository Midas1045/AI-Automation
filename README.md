# AN AUTOMATED STUDENT ONBOARDING SYSTEM
Training academies often handle student registration by hand: checking payment receipts, typing details into spreadsheets, adding students to classes, and sending welcome messages one by one. This is slow and prone to errors such as missed payments, duplicate records, and students waiting days for access.

An automated student onboarding system solves this by connecting the registration form, payment gateway, student database, learning platform, and email service into one workflow. When a student registers, the system validates their details, verifies their payment, creates or updates their record without duplication, enrolls them in the right course, and sends a welcome email. Failures, such as an unconfirmed payment, are retried and flagged to an admin.

This design shows how that process works, from the trigger to the final outcome, so the academy can onboard students faster, more accurately, and with less manual effort.

<p align="center"><img width="7201" height="3601" alt="Untitled Diagram12" src="https://github.com/user-attachments/assets/0981546d-13b8-45b1-8c40-0f11191d8d1d" />

## System/Workflow Explanation

1. Trigger: A student submits the registration form (Google Form or Typeform). That fires a webhook into the automation tool (n8n) 
2. Data fields: Full name, email, phone number, date, course selected alongside payment reference are captured here. If any are missing, the student is asked to resubmit
3. Payment Verification: The system calls the payment gateway API (Paystack) with the payment reference. It checks that the status is "success" and the amount matches the course fee. If not, the record is held,       the student is notified, the check retries again and the admin is alerted
4. Duplicate prevention: The system looks up the email and payment reference in the database. If the student exists, their record is updated. If not, a new record with a student ID is created.
5. Enrollment and email: The student is added to the course on the LMS, then a welcome email is sent with login details, start date and class link
6. APIs and webhooks: The form webhook is the inbound trigger. The payment gateway API is used for verification. The database, LMS and email service are outbound API calls.
7. Error handling: Failed payments, API timeouts and invalid data are retried and logged, with an admin alert if they still fail. A final log captures every outcome  
