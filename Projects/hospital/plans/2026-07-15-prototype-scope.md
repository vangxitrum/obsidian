---
type: plan
project: hospital
created: 2026-07-15
source: /home/tuan/personal/hospital/PROTOTYPE_SCOPE.md
---

# Tam Anh Smart Hospital Demo - Finalized Prototype Scope

## Purpose

Demonstration prototype of two connected services:
1. Smart hospital appointment booking and live queue tracking
2. Tam Anh HomeCare blood-test and vaccination booking

The prototype demonstrates user experience and operational value using seeded data and simulated integrations. Not a production medical system.

Source: [[file:///home/tuan/personal/hospital/PROTOTYPE_SCOPE.md|PROTOTYPE_SCOPE.md]]

## Core Demo Direction

- Demonstrate both appointment and HomeCare concepts
- Focus on one complete end-to-end journey per concept
- Patient-facing mobile application
- Hospital staff operations website
- Interactive simulations (no production integrations)
- Vietnamese as primary interface language

## Target Users

**Patients:**
- Adult patients booking for themselves
- Caregivers booking for children or elderly relatives
- Elderly and non-technical mobile users

**Staff:**
- Doctors managing live clinic queues
- HomeCare dispatch staff assigning nurses
- Laboratory staff publishing simulated results

## Patient Mobile Application

### Navigation
Bottom navigation with five destinations:
- Home
- Appointments
- HomeCare
- Notifications
- Account
- Persistent hotline/support action

### Onboarding & Account
- Branded splash screen
- Short onboarding introduction
- Vietnamese phone-number entry
- Simulated OTP verification
- Login persistence for demo session
- Basic patient profile
- Terms and privacy consent

### Family Profiles
- View, add, and select family members
- Store: name, DOB, gender, phone, relationship
- Display allergies or medical notes

## Appointment Booking Journey

### Discovery
- Browse short specialty list
- Search doctors by name
- Filter by specialty
- Doctor cards: photo, title, specialty, location, next availability
- Doctor profile with professional summary
- Consultation location
- Seeded available appointment slots

### Scheduling
- Select patient or family member
- Select hospital branch, date, time slot
- Enter short reason for visit
- Review booking details including full consultation fee and cancellation policy
- Hold selected slot for 10 minutes while payment pending
- Pay full displayed fee through MoMo Test
- Confirm booking only after verified payment success
- Clear payment and booking success screen
- Add only confirmed (paid) appointments to appointment list
- Generate digital ticket only after successful payment

### Appointment Management
- View upcoming appointments and details
- Cancel with confirmation dialog
- Explain that demo does not provide automatic refund after cancellation

### Digital Ticket
- Simulated queue number after booking
- Display: doctor, branch, room, date, time
- Personalized QR code
- Preparation and arrival instructions
- Ticket status

### Live Queue Tracking
- Current number being served
- Patient's queue number
- Number of patients ahead
- Estimated waiting time
- Simple queue progress indicator
- Real-time progress controlled from staff website
- Status text: `Chưa đến`, `Đã check-in`, `Đang chờ`, `Sắp đến lượt`, `Hoàn tất`
- Approaching-turn notification

### QR Check-in
- Display patient QR code
- Simulated "Check in now" action for demo
- Confirm successful check-in
- Update live queue state after check-in

## HomeCare Journey

### Service Catalogue
- Blood-test and vaccination categories
- Small set of seeded services
- Display: name, description, preparation requirements, duration, price
- Eligibility or age guidance where relevant

### HomeCare Booking
- Select blood test or vaccination service
- Select patient or family member
- Select saved home address or enter Vietnamese address details
- Choose seeded date and time window
- Enter access instructions and medical notes
- Review price breakdown including full service fee and cancellation policy
- Hold selected service window for 10 minutes while payment pending
- Pay full displayed fee through MoMo Test
- Confirm request only after verified payment success
- Clear payment and booking success screen
- Generate HomeCare confirmation only after successful payment

### Nurse Assignment & Tracking
- Simulated matching state
- Assigned nurse: name, photo, qualification, hospital ID
- Estimated arrival time
- Simulated map and nurse position
- Service status timeline
- Dispatch staff advance nurse status from website
- Arrival and delay notifications

### At-Home Service Status
- Patient verification as simulated QR event
- Service states: `Đã xác nhận`, `Đang di chuyển`, `Đã đến`, `Đang thực hiện`, `Hoàn tất`
- Simulated sample or vaccine batch code
- Aftercare instructions

### Results & Follow-up
- Publish one seeded blood-test result from staff website
- Notify patient when result ready
- Simple digital laboratory report
- Distinguish normal and attention-required values
- Medical disclaimer and hospital contact action
- One digital vaccination record
- Next recommended vaccination date

## Staff Operations Website

### Staff Access
- Simulated staff login
- Role selector for demo (Doctor, HomeCare dispatch, Laboratory)

### Doctor Queue Workspace
- Display today's seeded appointments
- Show checked-in and waiting patients
- Highlight current patient
- "Gọi bệnh nhân tiếp theo" action
- Mark consultation complete or patient absent
- Update patient mobile queue screen immediately
- Basic queue statistics

### HomeCare Dispatch Workspace
- Display pending and active HomeCare requests
- Simple map with seeded nurses and patients
- Assign nurse to request
- Advance nurse through simulated statuses
- Update patient mobile tracking screen immediately
- Highlight delayed visits

### Laboratory Workspace
- Display collected and processing samples
- Open seeded sample record
- Publish prepared result
- Update patient mobile result status immediately

### Demo Overview
- Concise dashboard: appointments, queue, HomeCare visits, result counts
- Clearly labeled simulated data

### Payment Visibility
- Show payment status beside appointment and HomeCare reservations
- Display MoMo order ID, transaction ID, amount, and payment time
- Keep unpaid reservations out of confirmed clinic queue and HomeCare schedule
- Show failed, cancelled, and expired payment attempts
- Do not provide refund action in demo

## Tam Anh Visual Direction

### Color Palette
- Primary navy: `#102E9E`
- Action blue: `#1691E2`
- Supporting blue: `#0B52CB`
- Soft blue background: `#F6F8FE`
- Neutral background: `#F0F2F1`
- Card background: `#FFFFFF`
- Primary text: `#333333`
- Secondary text: `#686868`
- Dividers: `#E9E9E9`
- Success green (completed states)
- Amber (waiting/attention states)
- Red (errors, cancellation, urgent warnings)
- Never communicate status through color alone

### Typography
- Primary typeface: Barlow
- Verify all Vietnamese diacritics
- Minimum 16px body text on mobile
- Prefer 18px for important instructions
- Clear, strong headings (not decorative)
- Maximum three font weights per screen

### Components
- White cards on soft blue/gray backgrounds
- Card corner radius: 12px-16px
- Restrained shadows and clear borders
- Solid blue primary buttons
- Outlined secondary buttons
- Large status chips with icon and text
- Familiar line icons with text labels
- Step indicators for multi-step booking
- Bottom sheets for short, focused choices
- Confirmation dialogs for destructive actions

### Imagery
- Professional Vietnamese doctor and nurse imagery
- Calm, clean hospital photography
- Real-looking profile photos consistently across demo
- Meaningful alternative text

## UX Rules for Non-Technical & Elderly Users

- One clear primary action per screen
- Touch targets minimum 48px × 48px
- Primary navigation visible and predictable
- No hidden swipe-only actions
- Avoid medical jargon; use plain Vietnamese
- Explain unfamiliar medical terms in one short sentence
- One group of related questions per step
- Pre-fill known patient and contact information
- Show progress during booking
- Review page before final confirmation
- Confirm successful actions with text, icon, and next step
- Preserve entered information after validation errors
- Error messages beside affected field
- Provide "Quay lại" and "Hủy" actions where appropriate
- Hotline support reachable within one tap from critical screens
- No automatic timeouts during forms in demo
- Screen-reader labels and logical focus order
- WCAG AA color contrast where practical
- Support system text scaling without hiding controls

## Vietnam Localization

- Vietnamese default language
- Natural, respectful Vietnamese copy
- Vietnamese names without assuming Western first/last name rules
- Vietnamese phone numbers: `09xx xxx xxx`
- Dates: `DD/MM/YYYY`
- Time: 24-hour format
- Prices in Vietnamese đồng: `850.000 ₫`
- Address fields: province/city, ward, street
- Seed Ho Chi Minh City Tam Anh locations
- Familiar terms: `Đặt lịch`, `Số thứ tự`, `Đang phục vụ`, `Còn ... người phía trước`
- Local hotline formatting

## Recommended Screen Inventory

### Patient Mobile (26 screens)
1. Splash and onboarding
2. Phone login and OTP
3. Home dashboard
4. Family profiles
5. Specialty list
6. Doctor list
7. Doctor profile
8. Appointment slot selection
9. Appointment review and payment
10. MoMo payment pending and countdown
11. Payment success
12. Payment failure
13. Payment cancellation
14. Payment expiration
15. Appointment confirmation
16. Digital ticket
17. Live queue
18. QR check-in confirmation
19. HomeCare service catalogue
20. HomeCare service details
21. HomeCare booking form
22. HomeCare booking review and payment
23. HomeCare confirmation after payment
24. Nurse matching and live tracking
25. Service status and aftercare
26. Test result
27. Vaccination record
28. Notifications
29. Account and support

### Staff Website (8 screens)
1. Staff login and demo role selection
2. Overview dashboard
3. Doctor queue workspace
4. HomeCare dispatch workspace
5. HomeCare request detail
6. Laboratory sample list
7. Laboratory result publication
8. Reservation payment details and status

## Demo Scripts

### Script A: Appointment & Queue
1. Patient logs in with Vietnamese phone number and simulated OTP
2. Patient selects family member
3. Patient chooses specialty and doctor
4. Patient chooses available time and app holds it for 10 minutes
5. Patient reviews full fee and cancellation policy
6. Patient completes payment through MoMo Test
7. Verified payment confirms booking
8. App generates queue number and QR ticket only after payment
9. Patient performs simulated QR check-in
10. Staff opens doctor queue website
11. Staff calls next patient
12. Patient's live queue screen updates
13. Patient receives approaching-turn notification

### Script B: HomeCare
1. Patient selects home blood-test or vaccination service
2. Patient selects family member, address, date, time window
3. App holds selected time window for 10 minutes
4. Patient reviews full price and cancellation policy
5. Patient completes payment through MoMo Test
6. Verified payment confirms HomeCare booking
7. Dispatch staff assigns seeded nurse
8. Patient sees nurse profile, ETA, map, status timeline
9. Dispatch staff advances nurse to `Đã đến` and `Hoàn tất`
10. Laboratory staff publishes seeded result
11. Patient receives notification and opens digital result

## MoMo Test Payment Sandbox

### Payment Rules
- Use MoMo Test for appointment and HomeCare reservations
- Use MoMo one-time wallet payment with `requestType: captureWallet`
- Charge full displayed fee in Vietnamese đồng
- Hold selected appointment slot or HomeCare time window for 10 minutes
- Display visible countdown while payment pending
- Confirm reservation only after verified successful payment notification
- Release held slot after payment failure, cancellation, or expiration
- Make payment creation and confirmation idempotent
- Payment states: `Chờ thanh toán`, `Đã thanh toán`, `Thanh toán thất bại`, `Đã hủy`, `Đã hết hạn`
- Reservation states: `Tạm giữ`, `Đã xác nhận`, `Đã hết hạn`, `Đã hủy`
- Show cancellation policy before patient starts payment
- No automatic or sandbox refunds included

### Patient Payment Experience
- Show payment review screen with patient, service, schedule, price, and cancellation policy
- Provide clear `Thanh toán bằng MoMo` action
- Open MoMo Test hosted checkout through payment URL or mobile deep link
- Return patient to application after checkout
- Show processing state while waiting for server confirmation
- Show dedicated success, failure, cancellation, and expiration states
- Allow failed or cancelled payment retry while hold remains valid
- Display paid amount, MoMo transaction reference, and payment time in reservation details
- Generate digital ticket or HomeCare confirmation only after successful payment

### Integration and Security
- Create payments through MoMo `/v2/gateway/api/create` endpoint
- Generate HMAC-SHA256 request signatures on server only
- Keep MoMo partner code, access key, and secret key outside client code and version control
- Use MoMo IPN as authoritative payment result (not redirect response)
- Verify IPN signature, partner code, order ID, request ID, and amount
- Treat only `resultCode = 0` as successful captured payment
- Return HTTP `204` after processing valid IPN request
- Handle duplicate and out-of-order payment notifications safely
- Query transaction status when payment confirmation delayed or uncertain
- Use HTTPS deployment or secure development tunnel for MoMo callback URL
- Never log payment secret keys or full sensitive callback payloads

## Simulated Demo Behavior

- Seed realistic Vietnamese patients, doctors, nurses, branches, specialties, services
- Persist demo state locally during session
- Allow demo reset to initial state
- Simulate OTP success with documented demo code
- Generate QR codes locally
- Simulate queue events from staff website
- Simulate nurse movement and status changes
- Simulate mobile push notifications inside app
- Simulate laboratory result publication
- Patient and staff interfaces share same demo state
- Connect to MoMo Test payment environment

## Explicitly Out of Scope

Production systems and integrations NOT included in baseline:
- Production clinical diagnosis or medical advice
- Real Hospital Information System integration
- Real laboratory integration
- Real-time doctor forecasting algorithms
- Real nurse dispatch optimization
- Real GPS tracking
- Production payment processing and automatic refunds
- Real push, SMS, email, or Zalo delivery
- Production identity verification
- Insurance claims
- Electronic medical records
- Prescription management
- IoT cold-chain devices
- Production security, compliance, and operational monitoring

## Prototype Acceptance Criteria

- Both demo scripts can be completed without dead ends
- Mobile and staff interfaces display consistent shared state
- Queue changes made on website appear in mobile experience
- HomeCare status changes made on website appear in mobile experience
- Published test results become visible in mobile experience
- Appointment and HomeCare reservations remain temporary until MoMo Test payment succeeds
- Verified successful payment confirms reservation exactly once
- Duplicate callbacks do not create duplicate reservations or payments
- Failed, cancelled, and expired payments release held slot
- Unpaid reservations expire after 10 minutes
- Invalid signatures, mismatched order IDs, and mismatched amounts are rejected
- Closing payment page does not lose successful payment confirmed by IPN
- Payment secrets never exposed to mobile application
- Complete demo can be reset quickly
- All screens work at common mobile and desktop demo sizes
- Core flows remain understandable without technical explanation
- Core actions usable with large text and large touch targets
- Vietnamese copy and diacritics reviewed before presentation
- No screen implies simulated data is live clinical data

## Demo Data & Safety

- Use fictional patient and staff identities
- Do not use real patient data
- Label simulated information where misunderstanding possible
- Add "Bản demo - Không dùng cho mục đích y tế" to appropriate screens
- Avoid presenting abnormal test results as diagnosis
- Direct users to contact hospital for medical interpretation
- Use Tam Anh name and logo only with appropriate permission
- Add prototype disclaimer if shown externally
