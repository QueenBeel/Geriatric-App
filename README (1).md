# Patient Assistant

A simple Android app that helps patients organize medications, receive reminders, record doses, and quickly reach an emergency contact.

**Status:** Planning and UI design. This README defines the proposed MVP; it does not represent a working application or completed features.

**Platform:** React Native + TypeScript, Android first. Eventual goal: Google Play publication.

## Product goal

Make everyday medication tracking easy, particularly for older adults. A user completes a short profile once, then returns directly to a clear home screen with four large actions.

The app records user-entered schedules and self-reported dose actions. It does not verify that medication was swallowed, recommend doses, check drug interactions, or automatically dispatch emergency help.

## MVP features

| Area | Planned behavior |
| --- | --- |
| First-time setup | Collect full name, phone number, emergency contact name, and emergency contact number. |
| Returning users | Load the saved profile and go directly to Home. |
| SOS / Emergency Help | Show the contact and open the phone dialer after an explicit tap. |
| Medication Reminders | Show today's doses and upcoming schedules. |
| Manage Medications | Add, view, edit, and remove medications. |
| Mark as Taken | Confirm a specific scheduled dose and record the action time. |
| Notifications | Local reminders with dose details, snooze, taken, and skipped actions. |
| Settings | Edit profile, manage reminder preferences, hide notification details, and delete all data. |

The first version uses one local profile per installation, works offline for core functions, and requires no account or backend. A saved phone number is not a verified identity. Calls depend on the device's calling capability and service.

## Screen flow

On startup, load local data before deciding which screen to show. Display a loading state while storage initializes; do not briefly show onboarding to returning users.

| Screen | Purpose / next action |
| --- | --- |
| Welcome | Introduce the app; Get Started opens profile setup. |
| Profile Setup | Validate and save required fields. |
| Reminder Permission | Explain notifications; allow Enable or Not Now. |
| Home | Greeting, date, next dose, Profile button, and the four main rows. |
| Emergency Help | Contact name, phone number, and Call Emergency Contact. |
| Reminders | Today's doses and their statuses; open dose details. |
| Medication List | View medications and open Add or Details. |
| Medication Form | Create or edit medication and schedule. |
| Medication Details | View instructions, edit, or confirm removal. |
| Dose Details | Mark as taken, snooze 10 minutes, or skip. |
| Profile / Settings | Edit contact information and preferences. |

Home rows, in order: **SOS / Emergency Help**, **Medication Reminders**, **Manage Medications**, and **Mark as Taken**. If multiple doses are due, Mark as Taken opens a selection list. If none are due, show “No medication is due right now.”

## Design direction

Create and review the Figma prototype before implementing screens.

- Light backgrounds, deep navy text, and teal primary actions.
- Red reserved for emergencies and important errors.
- Large, readable text; target 18–20 for body text and support system font scaling.
- Touch targets at least 48 × 48 dp; primary buttons preferably 56 dp tall.
- Rounded cards, restrained shadows, generous spacing, and visible labels.
- Icons plus text; never rely on color alone.
- Scrollable layouts that remain usable with enlarged text and narrow screens.
- Accessible labels, logical focus order, and Android TalkBack support.
- Persistent form labels, clear validation, and confirmation for destructive actions.

Design loading, empty, error, permission-denied, and success states. Use clearly labeled fictional medication examples in mockups.

## Proposed technical structure

Keep screens focused on presentation. Put persistence, notifications, scheduling, and dose actions behind separate interfaces so they can be tested independently.

| Layer | Proposed choice / responsibility |
| --- | --- |
| UI | React Native with TypeScript and reusable components. |
| Navigation | Typed screen navigation; select the navigation library during scaffolding. |
| State | Shared profile and medication state with explicit loading/error states. |
| Persistence | Local persistent storage, with a documented encryption and key-management approach. |
| Notifications | Android-capable local notification adapter, selected and verified during setup. |
| Scheduling | Pure date/time functions separate from UI and notification APIs. |
| Testing | Unit tests for scheduling and dose logic; device tests for Android behavior. |

Choose Expo development builds or a bare React Native project after verifying the chosen notification and secure-storage integrations. Pin compatible dependency versions in the actual project. No framework versions or setup commands are prescribed yet.

### Suggested folders

These paths are a proposed scaffold, not files already implemented.

| Path | Contents |
| --- | --- |
| `App.tsx` | App entry, providers, startup initialization. |
| `src/navigation/` | Root navigator, onboarding flow, screen route types. |
| `src/screens/onboarding/` | Welcome, ProfileSetup, ReminderPermission screens. |
| `src/screens/home/` | Home screen. |
| `src/screens/emergency/` | EmergencyHelp screen. |
| `src/screens/medications/` | MedicationList, MedicationForm, MedicationDetails screens. |
| `src/screens/reminders/` | Reminders, DoseSelection, DoseDetails screens. |
| `src/screens/settings/` | Profile and settings screens. |
| `src/components/` | ActionRow, MedicationCard, StatusBadge, FormField, PrimaryButton, ConfirmDialog. |
| `src/models/` | Profile, Medication, Schedule, DoseOccurrence, DoseAction, Settings types. |
| `src/services/storage/` | Repositories, schema migrations, and deletion logic. |
| `src/services/notifications/` | Permissions, scheduling, cancellation, and notification routing. |
| `src/services/emergency/` | Phone-number validation and dialer handoff. |
| `src/domain/` | Schedule generation, dose status, duplicate-action prevention. |
| `src/state/` | Shared state and app initialization. |
| `src/theme/` | Colors, typography, spacing, and component tokens. |
| `src/utils/` | Date formatting and general validation. |
| `assets/` | Icons and illustrations. |
| `tests/` | Logic, integration, and device-flow tests. |
| `docs/` | Design links, architecture decisions, and release notes. |

## Basic data models

| Model | Suggested fields |
| --- | --- |
| Profile | `id`, `fullName`, `phoneNumber`, `emergencyContactName`, `emergencyContactPhone`, `onboardingCompletedAt`, `createdAt`, `updatedAt` |
| Medication | `id`, `name`, `strength`, `amountPerDose`, `instructions?`, `isActive`, `createdAt`, `updatedAt` |
| Schedule | `id`, `medicationId`, `weekdays`, `localTimes`, `startDate`, `endDate?`, `timeZone`, `revision` |
| DoseOccurrence | `id`, `medicationId`, `scheduleId`, `scheduleRevision`, `scheduledAt`, medication/dose snapshot, `notificationIds`, `snoozedUntil?` |
| DoseAction | `id`, `doseOccurrenceId`, `action` (`taken` or `skipped`), `recordedAt`, `undoneAt?` |
| Settings | `hideMedicationInNotifications`, reminder preferences, optional text-size preference, `schemaVersion` |

Use stable IDs. Store absolute action timestamps and preserve the time-zone information needed to interpret schedules. Save a medication/dose snapshot on each occurrence so later edits do not rewrite history.

Derive Upcoming, Due, and Overdue / Not Recorded from the scheduled time and a documented due-window rule. Taken and Skipped come from explicit user actions. “Overdue / Not Recorded” means no action was recorded; it does not establish whether a patient took the medication.

## Core behavior rules

### Profile and medication management

- Require all four profile fields and validate phone numbers with country codes.
- Save onboarding completion only after the profile is stored successfully.
- Require medication name, strength, amount per dose, selected days, at least one time, and start date.
- Validate end dates, reject duplicate reminder times, and show useful field errors.
- Support daily or selected-weekday schedules in the MVP.
- Treat Remove as deactivation: cancel future reminders while preserving historical dose records.
- On schedule edits, replace future occurrences and notifications without changing past records.

### Dose actions and reminders

- Associate each notification and action with one stable dose-occurrence ID.
- Make confirmations idempotent: repeated taps must not create duplicate taken records.
- Provide Undo for an accidental confirmation and recompute the affected reminder state.
- Snoozing changes only the selected occurrence, not the recurring schedule.
- Cancel pending reminders for a dose after it is marked Taken or Skipped.
- Define and document the due window, occurrence-generation horizon, and time-zone policy before implementing scheduling.
- Reconcile reminders after startup, schedule edits, device restart, and relevant time changes; verify real Android behavior.
- Handle denied notification permissions and unavailable scheduling with a visible explanation and recovery action.
- Do not describe reminder delivery as guaranteed. Test locked-screen and background behavior on physical devices.

### Emergency action

- SOS opens an emergency screen; it does not place a call immediately.
- Call Emergency Contact opens the dialer with the saved number.
- Handle unavailable dialer apps and invalid or missing numbers.
- Do not claim a contact was notified, a call connected, or help was dispatched based on opening the dialer.

## Privacy and scope

- Keep health and contact information out of logs and analytics.
- Document storage encryption, key management, backup behavior, and any third-party data access before release.
- Offer generic notification text when medication-name privacy is enabled.
- Delete All My Data requires confirmation, cancels app reminders, clears local app data, and returns to onboarding.
- Request only permissions required by implemented features.

Outside the MVP: cloud synchronization, caregiver messaging, prescriptions, drug-interaction checks, dosage recommendations, automatic emergency alerts, and clinician integrations.

## Development roadmap

- [ ] Phase 1 — Create Figma screens, reusable components, and clickable flows.
- [ ] Phase 2 — Scaffold React Native/TypeScript, theme, and navigation.
- [ ] Phase 3 — Add persistent profile setup and returning-user routing.
- [ ] Phase 4 — Implement medication forms, schedules, and medication management.
- [ ] Phase 5 — Implement dose occurrences, history, taken/skip/undo actions.
- [ ] Phase 6 — Integrate local reminders, permission handling, and cancellation.
- [ ] Phase 7 — Add emergency contact dialer flow and settings.
- [ ] Phase 8 — Verify accessibility, privacy, Android behavior, and release readiness.

Build one working flow at a time. Start with Welcome → Profile Setup → Home, then medication management, then reminder behavior.

## Acceptance checklist

- [ ] Fresh installation opens onboarding; relaunch with a saved profile opens Home.
- [ ] Profile and medications survive app restarts.
- [ ] All four home actions work with readable text and large touch targets.
- [ ] Schedule edits/removal cancel stale future notifications.
- [ ] Multiple doses can be selected and recorded independently.
- [ ] Repeated confirmation cannot duplicate a dose action; Undo works.
- [ ] Snooze leaves the recurring schedule unchanged.
- [ ] Permission denial, restart, time-zone changes, and daylight-saving transitions are tested.
- [ ] Notification taps open the correct dose, including after app restart.
- [ ] SOS opens the correct dialer number only after a user tap.
- [ ] Enlarged text and TalkBack support the primary flows.
- [ ] Delete All My Data removes local records and pending app reminders.

## Running the project

There is no runnable scaffold yet. After project initialization, replace this section with verified prerequisites, dependency installation, Android emulator/device setup, development commands, test commands, and release-build commands from the actual repository. Do not treat this README as installation instructions for an existing app.

## Publication preparation

Before publication, verify the then-current Android and Google Play requirements against official documentation. Prepare the app identity, icon, screenshots, privacy policy, accurate data disclosures, signed release build, and required testing/review materials. Keep signing secrets outside source control.

**Design link:** To be added after the Figma prototype is created.

**License:** To be decided before distributing source code.
