# API Audit for Offline Support

This document lists some of the key backend APIs and workflows that need to be handled for robust offline support in the CARE application. The tables below provide an overview of standard workflows and the dynamic data that should be cached to ensure seamless user experience during connectivity interruptions.

These tables are intended to:
- Identify critical workflows and their associated APIs for offline operation
- Guide route registration in service workers (e.g., workbox)
- Inform caching and persistence strategies for dynamic data

## Standard Workflows and Associated APIs

| Workflow                | API Endpoint(s)                        | Description / Notes                                 |
|-------------------------|-----------------------------------------|-----------------------------------------------------|
| Patient Registration    | `/api/v1/patient/` (POST, GET)          | Create and fetch patient records                    |
| Patient Details         | `/api/v1/patient/{id}/` (GET, PUT)      | View and update patient details                     |
| Encounter Creation      | `/api/v1/encounter/` (POST, GET)        | Create and list encounters                          |
| Encounter Details       | `/api/v1/encounter/{id}/` (GET, PUT)    | View and update encounter details                   |
| Form Submission         | `/api/v1/form/` (POST, GET)             | Submit and fetch forms                              |
| Form Details            | `/api/v1/form/{id}/` (GET, PUT)         | View and update form details                        |
| Visit List              | `/api/v1/visit/` (GET)                  | List all visits for a patient                       |
| Visit Details           | `/api/v1/visit/{id}/` (GET, PUT)        | View and update visit details                       |
| User Profile            | `/api/v1/user/profile/` (GET, PUT)      | Fetch and update user profile                       |

## Notes
- The above APIs are examples; actual endpoints may vary based on backend implementation.
- These APIs should be registered for caching and persistence in the service worker to support offline workflows.
- Additional APIs may be added as new workflows are identified for offline support.

## How to Use This Audit
- Use this list to inform which routes should be registered in the workbox/service worker for caching.
- Reference this document when planning or updating offline support features and CEPs. 