Client flow:

/book-online:
1. bookings/v2/services/query -> respose is a list of services:
(open question: one refresh this call doesnt happens only when going to some other page and back - understand in the client when and how they do it)
{
    "services": [
        # COURSE type service
        {
            "id": "cd3eb411-3f95-4a13-b529-7134330df50c",
            "appId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
            "createdByAppId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
            "type": "COURSE",
            "sortOrder": 5,
            "name": "Course No Staff",
            "defaultCapacity": 100,
            "media": {
                "items": [],
                "mainMedia": {
                    "image": {
                        "id": "1ea33919-a304-49c3-95a6-8d3aac48bb81",
                        "url": "11062b_1ea33919a30449c395a68d3aac48bb81~mv2.png",
                        "height": 3468,
                        "width": 3746,
                        "filename": "11062b_1ea33919a30449c395a68d3aac48bb81~mv2.png"
                    }
                }
            },
            "hidden": false,
            "category": {
                "id": "1e4fa8e8-96e7-40a6-a07e-f215083a9ded",
                "name": "Our Services",
                "sortOrder": 2054728
            },
            "form": {
                "id": "00000000-0000-0000-0000-000000000000",
                "mobileSettings": {
                    "hidden": false
                }
            },
            "payment": {
                "rateType": "FIXED",
                "fixed": {
                    "price": {
                        "value": "180",
                        "currency": "USD"
                    }
                },
                "options": {
                    "online": false,
                    "inPerson": true,
                    "deposit": false
                },
                "pricingPlanIds": [],
                "addOnOption": "UNKNOWN_ADD_ON_PAYMENT_OPTIONS"
            },
            "onlineBooking": {
                "enabled": true,
                "requireManualApproval": false,
                "allowMultipleRequests": false
            },
            "conferencing": {
                "enabled": false
            },
            "locations": [
                {
                    "id": "237266d6-22db-4346-805e-c3feaab1826c",
                    "type": "BUSINESS",
                    "calculatedAddress": {
                        "country": "US"
                    },
                    "business": {
                        "id": "237266d6-22db-4346-805e-c3feaab1826c",
                        "name": "Location 1",
                        "default": true,
                        "address": {
                            "country": "US"
                        }
                    }
                }
            ],
            "bookingPolicy": {
                "id": "7a0fdb27-9c4b-4911-9cbe-9f8cb07694d6",
                "revision": "1",
                "createdDate": "2025-12-21T10:03:44.974Z",
                "updatedDate": "2025-12-21T10:03:44.974Z",
                "appId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
                "name": "Default policy",
                "customPolicyDescription": {
                    "enabled": true,
                    "description": ""
                },
                "default": true,
                "limitEarlyBookingPolicy": {
                    "enabled": false,
                    "earliestBookingInMinutes": 10080
                },
                "limitLateBookingPolicy": {
                    "enabled": false,
                    "latestBookingInMinutes": 1440
                },
                "bookAfterStartPolicy": {
                    "enabled": false
                },
                "cancellationPolicy": {
                    "enabled": true,
                    "limitLatestCancellation": false,
                    "latestCancellationInMinutes": 1440
                },
                "reschedulePolicy": {
                    "enabled": true,
                    "limitLatestReschedule": false,
                    "latestRescheduleInMinutes": 1440
                },
                "waitlistPolicy": {
                    "enabled": false,
                    "capacity": 10,
                    "reservationTimeInMinutes": 10
                },
                "participantsPolicy": {
                    "enabled": true,
                    "maxParticipantsPerBooking": 1
                },
                "resourcesPolicy": {
                    "enabled": false,
                    "autoAssignAllowed": false
                },
                "cancellationFeePolicy": {
                    "enabled": false,
                    "cancellationWindows": [],
                    "autoCollectFeeEnabled": true
                },
                "saveCreditCardPolicy": {
                    "enabled": false
                },
                "staffSortingPolicy": {
                    "sortingMethodType": "RANDOM"
                }
            },
            "schedule": {
                "id": "847af652-1833-42df-9fe6-0dbdfd8ced0e",
                "firstSessionStart": "2026-01-21T15:30:00Z",
                "lastSessionEnd": "2026-04-21T15:30:00Z",
                "availabilityConstraints": {
                    "durations": [
                        {
                            "minutes": 60
                        }
                    ],
                    "sessionDurations": [
                        60
                    ],
                    "timeBetweenSessions": 0
                }
            },
            "staffMemberIds": [
                "62934c62-004f-4cef-b3ac-07e2050afcb7"
            ],
            "staffMembers": [
                {
                    "staffMemberId": "62934c62-004f-4cef-b3ac-07e2050afcb7",
                    "name": "Business Owner",
                    "workingHoursScheduleIds": [
                        "82c29127-c187-4f64-ae1b-538a62ef825a"
                    ],
                    "eventsScheduleId": "d939f710-c234-41f0-8b15-d2e87febb80d"
                }
            ],
            "staffMemberDetails": {
                "staffMembers": [
                    {
                        "staffMemberId": "62934c62-004f-4cef-b3ac-07e2050afcb7",
                        "name": "Business Owner",
                        "workingHoursScheduleIds": [
                            "82c29127-c187-4f64-ae1b-538a62ef825a"
                        ],
                        "eventsScheduleId": "d939f710-c234-41f0-8b15-d2e87febb80d"
                    }
                ]
            },
            "resourceGroups": [],
            "serviceResources": [],
            "supportedSlugs": [
                {
                    "name": "course-no-staff",
                    "custom": false,
                    "createdDate": "2025-12-31T15:27:46.714Z"
                },
                {
                    "name": "copy-of-course-1-staff-1",
                    "custom": false,
                    "createdDate": "2025-12-31T15:27:24.858Z"
                }
            ],
            "mainSlug": {
                "name": "course-no-staff",
                "custom": false,
                "createdDate": "2025-12-31T15:27:46.714Z"
            },
            "urls": {
                "servicePage": {
                    "relativePath": "/service-page/course-no-staff",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/service-page/course-no-staff"
                },
                "bookingPage": {
                    "relativePath": "/service-page/course-no-staff",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/service-page/course-no-staff"
                }
            },
            "createdDate": "2025-12-31T15:27:24.859Z",
            "updatedDate": "2025-12-31T18:33:50.681Z",
            "revision": "5",
            "addOnGroups": [],
            "addOnDetails": [],
            "taxableAddress": {
                "taxableAddressType": "BUSINESS"
            }
        },
        #################### CLASS type service
        {
            "id": "512eb85b-61cd-4397-ae5a-1b698c8b32af",
            "appId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
            "createdByAppId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
            "type": "CLASS",
            "sortOrder": 11,
            "name": "Yoga Flow",
            "tagLine": "Improve flexibility and reduce stress through dynamic yoga movements",
            "defaultCapacity": 10,
            "media": {
                "items": [],
                "mainMedia": {
                    "image": {
                        "id": "11062b_82adfddb586c4fb9a4d2461c729df5c8~mv2.jpg",
                        "url": "11062b_82adfddb586c4fb9a4d2461c729df5c8~mv2.jpg",
                        "height": 0,
                        "width": 0,
                        "filename": "11062b_82adfddb586c4fb9a4d2461c729df5c8~mv2.jpg"
                    }
                }
            },
            "hidden": false,
            "category": {
                "id": "1e4fa8e8-96e7-40a6-a07e-f215083a9ded",
                "name": "Our Services",
                "sortOrder": 2054728
            },
            "form": {
                "id": "00000000-0000-0000-0000-000000000000",
                "mobileSettings": {
                    "hidden": true
                }
            },
            "payment": {
                "rateType": "FIXED",
                "fixed": {
                    "price": {
                        "value": "15",
                        "currency": "USD"
                    }
                },
                "options": {
                    "online": false,
                    "inPerson": true,
                    "deposit": false,
                    "pricingPlan": false
                },
                "pricingPlanIds": [],
                "addOnOption": "UNKNOWN_ADD_ON_PAYMENT_OPTIONS"
            },
            "onlineBooking": {
                "enabled": true,
                "requireManualApproval": false,
                "allowMultipleRequests": false
            },
            "conferencing": {
                "enabled": false
            },
            "locations": [
                {
                    "id": "237266d6-22db-4346-805e-c3feaab1826c",
                    "type": "BUSINESS",
                    "calculatedAddress": {
                        "country": "US"
                    },
                    "business": {
                        "id": "237266d6-22db-4346-805e-c3feaab1826c",
                        "name": "Location 1",
                        "default": true,
                        "address": {
                            "country": "US"
                        }
                    }
                }
            ],
            "bookingPolicy": {
                "id": "7a0fdb27-9c4b-4911-9cbe-9f8cb07694d6",
                "revision": "1",
                "createdDate": "2025-12-21T10:03:44.974Z",
                "updatedDate": "2025-12-21T10:03:44.974Z",
                "appId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
                "name": "Default policy",
                "customPolicyDescription": {
                    "enabled": true,
                    "description": ""
                },
                "default": true,
                "limitEarlyBookingPolicy": {
                    "enabled": false,
                    "earliestBookingInMinutes": 10080
                },
                "limitLateBookingPolicy": {
                    "enabled": false,
                    "latestBookingInMinutes": 1440
                },
                "bookAfterStartPolicy": {
                    "enabled": false
                },
                "cancellationPolicy": {
                    "enabled": true,
                    "limitLatestCancellation": false,
                    "latestCancellationInMinutes": 1440
                },
                "reschedulePolicy": {
                    "enabled": true,
                    "limitLatestReschedule": false,
                    "latestRescheduleInMinutes": 1440
                },
                "waitlistPolicy": {
                    "enabled": false,
                    "capacity": 10,
                    "reservationTimeInMinutes": 10
                },
                "participantsPolicy": {
                    "enabled": true,
                    "maxParticipantsPerBooking": 1
                },
                "resourcesPolicy": {
                    "enabled": false,
                    "autoAssignAllowed": false
                },
                "cancellationFeePolicy": {
                    "enabled": false,
                    "cancellationWindows": [],
                    "autoCollectFeeEnabled": true
                },
                "saveCreditCardPolicy": {
                    "enabled": false
                },
                "staffSortingPolicy": {
                    "sortingMethodType": "RANDOM"
                }
            },
            "schedule": {
                "id": "35beb0bb-37b8-4e2f-90c3-5585078425aa",
                "firstSessionStart": "2026-01-08T21:30:00Z",
                "availabilityConstraints": {
                    "durations": [
                        {
                            "minutes": 60
                        }
                    ],
                    "sessionDurations": [
                        60
                    ],
                    "timeBetweenSessions": 0
                }
            },
            "staffMemberIds": [
                "62934c62-004f-4cef-b3ac-07e2050afcb7"
            ],
            "staffMembers": [
                {
                    "staffMemberId": "62934c62-004f-4cef-b3ac-07e2050afcb7",
                    "name": "Business Owner",
                    "workingHoursScheduleIds": [
                        "82c29127-c187-4f64-ae1b-538a62ef825a"
                    ],
                    "eventsScheduleId": "d939f710-c234-41f0-8b15-d2e87febb80d"
                }
            ],
            "staffMemberDetails": {
                "staffMembers": [
                    {
                        "staffMemberId": "62934c62-004f-4cef-b3ac-07e2050afcb7",
                        "name": "Business Owner",
                        "workingHoursScheduleIds": [
                            "82c29127-c187-4f64-ae1b-538a62ef825a"
                        ],
                        "eventsScheduleId": "d939f710-c234-41f0-8b15-d2e87febb80d"
                    }
                ]
            },
            "resourceGroups": [],
            "serviceResources": [],
            "supportedSlugs": [
                {
                    "name": "yoga-flow",
                    "custom": false,
                    "createdDate": "2026-01-07T09:21:57.720Z"
                }
            ],
            "mainSlug": {
                "name": "yoga-flow",
                "custom": false,
                "createdDate": "2026-01-07T09:21:57.720Z"
            },
            "urls": {
                "servicePage": {
                    "relativePath": "/service-page/yoga-flow",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/service-page/yoga-flow"
                },
                "bookingPage": {
                    "relativePath": "/booking-calendar/yoga-flow",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/booking-calendar/yoga-flow"
                },
                "calendarPage": {
                    "relativePath": "/booking-calendar/yoga-flow",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/booking-calendar/yoga-flow"
                }
            },
            "createdDate": "2026-01-07T09:21:57.970Z",
            "updatedDate": "2026-01-07T09:22:36.968Z",
            "revision": "2",
            "addOnGroups": [],
            "addOnDetails": [],
            "taxableAddress": {
                "taxableAddressType": "BUSINESS"
            }
        }
    ],
    "pagingMetadata": {
        "count": 2,
        "offset": 0,
        "total": 2,
        "hasNext": false
    }
}

2.
for class services, theres another call:
(open question: is this call needed to get the data to apply the bookingsPolicies from (1))
for course services:
request: https://tamirc81.wixsite.com/_api/schedule-reader-server/v1/calendar/schedules?scheduleOwnerIds=cd3eb411-3f95-4a13-b529-7134330df50c&includeTotalNumberOfParticipants=true (https://github.com/wix-private/scheduler/blob/caf5ca4068a0639ed4b154ef7a059d27e5a6a489/bookings-backend/calendar-3/schedules/src/com/wixpress/calendar/schedules/v3/SchedulesService.scala#L63)

response (https://github.com/wix-private/scheduler/blob/bb11ae7507146a195043707f56ad09fe01aa2f69/bookings-backend/calendar-3/schedules/proto/wix/calendar/schedules/v3/schedules_service.proto#L372):

{
    "schedules": [
        {
            "id": "847af652-1833-42df-9fe6-0dbdfd8ced0e",
            "scheduleOwnerId": "cd3eb411-3f95-4a13-b529-7134330df50c",
            "firstSessionStart": "2026-01-21T15:30:00Z",
            "lastSessionEnd": "2026-04-21T15:30:00Z",
            "timeZone": "America/Chicago",
            "intervals": [],
            "title": "Course No Staff",
            "tags": [
                "COURSE"
            ],
            "location": {
                "locationType": "OWNER_BUSINESS",
                "businessLocation": {
                    "id": "237266d6-22db-4346-805e-c3feaab1826c",
                    "name": "Location 1",
                    "default": false,
                    "status": "ACTIVE",
                    "locationType": "UNKNOWN",
                    "archived": false,
                    "locationTypes": []
                }
            },
            "capacity": 100,
            "availability": {
                "start": "2025-12-31T15:27:24.745Z",
                "linkedSchedules": [],
                "locations": [],
                "useDefaultLocation": true
            },
            "totalNumberOfParticipants": 1,
            "participants": [],
            "status": "CREATED",
            "created": "2025-12-31T15:27:24.745Z",
            "updated": "2025-12-31T15:28:40.670Z",
            "version": 3,
            "versions": {
                "scheduleVersion": 3
            },
            "inheritedFields": [
                "timeZone"
            ],
            "scheduleOwnerName": "Course No Staff"
        }
    ]
}

client uses this to present if the class "book now" button is enabled and show more details in the widget.
after user press the "book now" button client redirects to service-page.



## /service-page:
for course:
(open question: does this calls and funtionalty changes depends on if the course is available for booking)
(open question: does the booking policies enforcments calculation happens here again? if so how costly is this proccess?)

1. request (get the cousre service object and bookingPolicies): 'https://tamirc81.wixsite.com/_api/bookings/v2/services/query'
  --data-raw '{"conditionalFields":["STAFF_MEMBER_DETAILS"],"query":{"filter":{"hidden":false,"supportedSlugs.name":"ai-with-tamir"}}}'

response:
{
    "services": [
        {
            "id": "5c09be59-0972-4851-aff1-cedf823011ca",
            "appId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
            "createdByAppId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
            "type": "COURSE",
            "sortOrder": 3,
            "name": "AI With Tamir",
            "defaultCapacity": 5,
            "media": {
                "items": [],
                "mainMedia": {
                    "image": {
                        "id": "1ea33919-a304-49c3-95a6-8d3aac48bb81",
                        "url": "11062b_1ea33919a30449c395a68d3aac48bb81~mv2.png",
                        "height": 3468,
                        "width": 3746,
                        "filename": "11062b_1ea33919a30449c395a68d3aac48bb81~mv2.png"
                    }
                }
            },
            "hidden": false,
            "category": {
                "id": "1e4fa8e8-96e7-40a6-a07e-f215083a9ded",
                "name": "Our Services",
                "sortOrder": 2054728
            },
            "form": {
                "id": "00000000-0000-0000-0000-000000000000",
                "mobileSettings": {
                    "hidden": false
                }
            },
            "payment": {
                "rateType": "FIXED",
                "fixed": {
                    "price": {
                        "value": "180",
                        "currency": "USD"
                    }
                },
                "options": {
                    "online": false,
                    "inPerson": true,
                    "deposit": false
                },
                "pricingPlanIds": [],
                "addOnOption": "UNKNOWN_ADD_ON_PAYMENT_OPTIONS"
            },
            "onlineBooking": {
                "enabled": true,
                "requireManualApproval": false,
                "allowMultipleRequests": false
            },
            "conferencing": {
                "enabled": false
            },
            "locations": [
                {
                    "id": "237266d6-22db-4346-805e-c3feaab1826c",
                    "type": "BUSINESS",
                    "calculatedAddress": {
                        "country": "US"
                    },
                    "business": {
                        "id": "237266d6-22db-4346-805e-c3feaab1826c",
                        "name": "Location 1",
                        "default": true,
                        "address": {
                            "country": "US"
                        }
                    }
                }
            ],
            "bookingPolicy": {
                "id": "7a0fdb27-9c4b-4911-9cbe-9f8cb07694d6",
                "revision": "5",
                "createdDate": "2025-12-21T10:03:44.974Z",
                "updatedDate": "2026-01-07T15:49:52.050Z",
                "appId": "13d21c63-b5ec-5912-8397-c3a5ddb27a97",
                "name": "Default policy",
                "customPolicyDescription": {
                    "enabled": true,
                    "description": ""
                },
                "default": true,
                "limitEarlyBookingPolicy": {
                    "enabled": false,
                    "earliestBookingInMinutes": 10080
                },
                "limitLateBookingPolicy": {
                    "enabled": false,
                    "latestBookingInMinutes": 21600
                },
                "bookAfterStartPolicy": {
                    "enabled": false
                },
                "cancellationPolicy": {
                    "enabled": true,
                    "limitLatestCancellation": false,
                    "latestCancellationInMinutes": 1440
                },
                "reschedulePolicy": {
                    "enabled": true,
                    "limitLatestReschedule": false,
                    "latestRescheduleInMinutes": 1440
                },
                "waitlistPolicy": {
                    "enabled": false,
                    "capacity": 10,
                    "reservationTimeInMinutes": 10
                },
                "participantsPolicy": {
                    "enabled": true,
                    "maxParticipantsPerBooking": 1
                },
                "resourcesPolicy": {
                    "enabled": false,
                    "autoAssignAllowed": false
                },
                "cancellationFeePolicy": {
                    "enabled": false,
                    "cancellationWindows": [],
                    "autoCollectFeeEnabled": true
                },
                "saveCreditCardPolicy": {
                    "enabled": false
                },
                "staffSortingPolicy": {
                    "sortingMethodType": "RANDOM"
                }
            },
            "schedule": {
                "id": "01ebfc3c-8044-4dd1-a91a-705cf242dad5",
                "firstSessionStart": "2026-01-07T15:00:00Z",
                "lastSessionEnd": "2026-04-29T17:30:00Z",
                "availabilityConstraints": {
                    "durations": [
                        {
                            "minutes": 60
                        }
                    ],
                    "sessionDurations": [
                        60
                    ],
                    "timeBetweenSessions": 0
                }
            },
            "staffMemberIds": [
                "62934c62-004f-4cef-b3ac-07e2050afcb7",
                "b328e6c1-31d0-4446-9cdb-87beeca56704"
            ],
            "staffMembers": [
                {
                    "staffMemberId": "62934c62-004f-4cef-b3ac-07e2050afcb7",
                    "name": "Business Owner",
                    "workingHoursScheduleIds": [
                        "82c29127-c187-4f64-ae1b-538a62ef825a"
                    ],
                    "eventsScheduleId": "d939f710-c234-41f0-8b15-d2e87febb80d"
                },
                {
                    "staffMemberId": "b328e6c1-31d0-4446-9cdb-87beeca56704",
                    "name": "Tamir",
                    "workingHoursScheduleIds": [
                        "82c29127-c187-4f64-ae1b-538a62ef825a"
                    ],
                    "eventsScheduleId": "1cadce73-5687-42b2-b970-c6e4c29d04db"
                }
            ],
            "staffMemberDetails": {
                "staffMembers": [
                    {
                        "staffMemberId": "62934c62-004f-4cef-b3ac-07e2050afcb7",
                        "name": "Business Owner",
                        "workingHoursScheduleIds": [
                            "82c29127-c187-4f64-ae1b-538a62ef825a"
                        ],
                        "eventsScheduleId": "d939f710-c234-41f0-8b15-d2e87febb80d"
                    },
                    {
                        "staffMemberId": "b328e6c1-31d0-4446-9cdb-87beeca56704",
                        "name": "Tamir",
                        "workingHoursScheduleIds": [
                            "82c29127-c187-4f64-ae1b-538a62ef825a"
                        ],
                        "eventsScheduleId": "1cadce73-5687-42b2-b970-c6e4c29d04db"
                    }
                ]
            },
            "resourceGroups": [],
            "serviceResources": [],
            "supportedSlugs": [
                {
                    "name": "ai-with-tamir",
                    "custom": false,
                    "createdDate": "2025-12-29T16:36:47.926Z"
                },
                {
                    "name": "copy-of-ai-with-tamir",
                    "custom": false,
                    "createdDate": "2025-12-25T11:45:36.786Z"
                }
            ],
            "mainSlug": {
                "name": "ai-with-tamir",
                "custom": false,
                "createdDate": "2025-12-29T16:36:47.926Z"
            },
            "urls": {
                "servicePage": {
                    "relativePath": "/service-page/ai-with-tamir",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/service-page/ai-with-tamir"
                },
                "bookingPage": {
                    "relativePath": "/service-page/ai-with-tamir",
                    "url": "https://tamirc81.wixsite.com/tamirs-ai-academy/service-page/ai-with-tamir"
                }
            },
            "createdDate": "2025-12-25T11:45:36.787Z",
            "updatedDate": "2026-01-07T15:56:29.965Z",
            "revision": "6",
            "addOnGroups": [],
            "addOnDetails": [],
            "taxableAddress": {
                "taxableAddressType": "BUSINESS"
            }
        }
    ],
    "pagingMetadata": {
        "count": 1,
        "cursors": {},
        "hasNext": false
    }
}

2. use the data from step above to query the schedule object of this course service.  
request: schedule-server/v1/calendar/schedules?scheduleOwnerIds=5c09be59-0972-4851-aff1-cedf823011ca&includeTotalNumberOfParticipants=true

response:
{
    "schedules": [
        {
            "id": "01ebfc3c-8044-4dd1-a91a-705cf242dad5",
            "scheduleOwnerId": "5c09be59-0972-4851-aff1-cedf823011ca",
            "firstSessionStart": "2026-01-07T15:00:00Z",
            "lastSessionEnd": "2026-04-29T17:30:00Z",
            "timeZone": "America/Chicago",
            "intervals": [],
            "title": "AI With Tamir",
            "tags": [
                "COURSE"
            ],
            "location": {
                "locationType": "OWNER_BUSINESS",
                "businessLocation": {
                    "id": "237266d6-22db-4346-805e-c3feaab1826c",
                    "name": "Location 1",
                    "default": false,
                    "status": "ACTIVE",
                    "locationType": "UNKNOWN",
                    "archived": false,
                    "locationTypes": []
                }
            },
            "capacity": 5,
            "availability": {
                "start": "2025-12-25T11:45:36.664Z",
                "linkedSchedules": [],
                "locations": [],
                "useDefaultLocation": true
            },
            "totalNumberOfParticipants": 2,
            "participants": [],
            "status": "CREATED",
            "created": "2025-12-25T11:45:36.664Z",
            "updated": "2025-12-29T16:36:48.275Z",
            "version": 2,
            "versions": {
                "scheduleVersion": 2
            },
            "inheritedFields": [
                "timeZone"
            ],
            "scheduleOwnerName": "AI With Tamir"
        }
    ]
}

3. query all the events of the schudale in order to present them:
request:
curl 'https://tamirc81.wixsite.com/_api/calendar/v3/events/query'
  --data-raw '{"fromLocalDate":"2026-01-07T09:00:00","toLocalDate":"2026-04-29T12:30:00","query":{"filter":{"$and":[{"appId":"13d21c63-b5ec-5912-8397-c3a5ddb27a97"},{"scheduleId":"01ebfc3c-8044-4dd1-a91a-705cf242dad5"}]}}}'

resposne: list of all events objects
[https://github.com/wix-private/scheduler/blob/28f20590c51cd616ea01545d258ce411d6315467/bookings-backend/calendar-3/events/proto/wix/calendar/events/v3/events_service.proto#L744]


for class:
(open question: does this calls and funtionalty changes depends on if the course is available for booking)
(open question: does the booking policies enforcments calculation happens here again? if so how costly is this proccess?)

1. same as (1) for course - query the service object of the class

2. get the list of all time slots of the events in the schudale 
request: [https://github.com/wix-private/scheduler/blob/77e543585a5d124423f188772258efc320ce7ffb/bookings-backend/service-availability/src/com/wixpress/bookings/availability/v2/EventTimeSlots.scala#L43]
curl 'https://tamirc81.wixsite.com/_api/service-availability/v2/time-slots/event' \
  --data-raw '{"timeZone":"America/Chicago","fromLocalDate":"2026-01-07T11:01:07","toLocalDate":"2026-01-31T23:59:59","serviceIds":["512eb85b-61cd-4397-ae5a-1b698c8b32af"],"maxSlotsPerDay":1,"includeNonBookable":true,"eventFilter":{},"cursorPaging":{"limit":1000}}'

response:
[https://github.com/wix-private/scheduler/blob/a50b6eb2143f0496aaef66b05c2a9006d5d4dd3e/bookings-backend/service-availability/proto/com/wixpress/bookings/availability/v2/event_time_slots.proto#L169]

list of time slots of the events (open question: i think here some of the bookingPolicies enforcment are done in the beckend but is it all of them? if not what is calculated in the client?)

* here if a user selects in the calander view to peresent given day we will request the events time slots for this dya using the same api with different request filter (for example):
curl 'https://tamirc81.wixsite.com/_api/service-availability/v2/time-slots/event' \
  --data-raw '{"timeZone":"America/Chicago","fromLocalDate":"2026-01-15T00:00:00","toLocalDate":"2026-01-15T23:59:59","serviceIds":["512eb85b-61cd-4397-ae5a-1b698c8b32af"],"includeNonBookable":true,"eventFilter":{},"cursorPaging":{"limit":1000}}'



tasks for you to add here:
we want to add support for course in the service-availability/v2/time-slots/event - same as we currently have for classes in the   override def listEventTimeSlots(request: ListEventTimeSlotsRequest) function
at bookings-backend/service-availability/src/com/wixpress/bookings/availability/v2/EventTimeSlots.scala 

for that i need you to:
1. trace the full logic and functionaly of a requst for classes, create flow chart of desicion logic and sequnce diagrams, explain the full proccess, which extarnel api are used, what calculation is made, how request filter effects the response, etc...

2. describe what will happend today if we send a request where serviceIds is a course service id, what will happend, what will the response look like, what filter is supported, etc...

3. conduct a comparnace table between 1 (class) and 2 (course) to indentify what is missing for course.

take into account that in class each timeSlot can have its own data - they do not have to be booked togather, a user can be booked to some of the timeSlots (events) of the class, in comparance this is not the case in course, in a course a user can only signup for all o the events (timeSlots) of the course meaning that after we calculated "totalCapacity", "remainingCapacity", "bookableCapacity", "bookingPolicyViolations", "bookable" and maybe some other shared fields for the first event (timeSlots) we do not want to calc it again just apply it to all of hte timeSlots in the response.

create a full system design propusal draft to implement this change and describe it using flow chart of desicion logic and sequnce diagrams, what benefits are we getting, what are the cons, what open questions needs to be answered and how they effect the design, etc... try to keep it clean and simple as it is the first draft