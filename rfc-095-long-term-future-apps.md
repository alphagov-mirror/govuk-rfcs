# RFC 95: Long term future of applications

## Summary

This RFC describes the direction of travel for the applications that make up GOV.UK.

## Problem

We currently - September 2018 - have 58 separate applications running on GOV.UK. The [Platform architecture principles][princ] explain where we're going from an architectural standpoint. This RFC is an attempt to explain the long term vision of each of GOV.UK's apps.

The purpose of doing this is to make clear:

- where to put new functionality
- which applications need particular attention

## Proposal

| Name | Description | Rough plan |
| -- | -- | -- |
| calculators | Serves the Child benefit tax calculator on GOV.UK | To be merged into smart-answers |
| calendars | Serves /bank-holidays and /when-do-the-clocks-change on GOV.UK | To be merged into custom frontend |
| ckanext-datagovuk | Extension for use with datagovuk_publish | Unknown |
| collections-publisher | Publishes step by steps, /browse pages, and legacy /topic pages on GOV.UK | While topics and browse pages will be removed, this can still be the app for curating collections. We might move Whitehall's document collections into this app  |
| content-audit-tool |  | Needs to be retired |
| content-performance-manager | Data warehouse that stores content and content metrics to help content owners measure and improve content on GOV.UK | Is currently a data warehouse, needs to be renamed |
| content-publisher | WIP - Future publisher of content on GOV.UK | Is the future of all publishing |
| datagovuk_find | Beta version of Find Data | Future unknown |
| datagovuk_publish | Beta version of publish data | Future unknown |
| email-alert-service | Message queue consumer that triggers email alerts for GOV.UK | Can be merged into email-alert-api, as it's only task is to take things from the message queue and forward it there |
| finder-frontend | Serves search pages for GOV.UK | No change expected. Can be renamed to "search-frontend". |
| frontend | Serves the homepage, transactions and some index pages on GOV.UK | To be retired. |
| info-frontend | Serves /info pages to display user needs and performance data about a page on GOV.UK | No change expected |
| licence-finder | Serves licence pages on GOV.UK | ?? |
| licensify | GOV.UK Licensing (formerly ELMS, Licence Application Tool, & Licensify) | Likely to be retired |
| manuals-frontend | Serves manuals on GOV.UK | Could be merged into government-frontend |
| manuals-publisher | Publishes manuals on GOV.UK | Should be retired in favour of content publisher |
| mapit | GOV.UK fork of Mapit, a web service to map postcodes to administrative boundaries |  |
| organisations-publisher | (Work in progress) Application for managing organisations, people and roles within GOV.UK | Will take on the organisation and "machinery of government" functionality of whitehall |
| policy-publisher | Publishes policies on GOV.UK | Will be retired, as the policy format will be retired |
| publisher | Publishes mainstream content on GOV.UK | Should be retired in favour of content publisher |
| router-api | API for updating the routes used by the router on GOV.UK | Functionality should be merged into either content store or publishing-api |
| rummager | Search API for GOV.UK | Should be renamed to search-api. |
| service-manual-frontend | Serves the Service Manual and Service Toolkit on GOV.UK | Parts of the frontend could be merged into government-frontend, and other apps. |
| service-manual-publisher | Publishes the Service Manual on GOV.UK | Custom application that could long term be replaced by content-publisher  |
| specialist-publisher | Publishes specialist documents on GOV.UK | Should be retired in favour of content publisher |
| static | GOV.UK static files and resources | Will be retired in favour of the components gem |
| travel-advice-publisher | Publishes foreign travel advice on GOV.UK | Should be retired in favour of content publisher |
| whitehall | Publishes government content on GOV.UK | Should be retired in favour of content publisher, organisations publisher, and collections publisher |

### No changes expected

| Name | Description | Rough plan |
| -- | -- | -- |
| asset-manager | Manages uploaded assets (images, PDFs etc.) for applications on GOV.UK | No change expected |
| authenticating-proxy | Allows authorised users to access the GOV.UK draft stack | No change expected |
| bouncer | Handles traffic for sites that have transitioned to GOV.UK | No change expected |
| cache-clearing-service | Clears various caches when new content is published. | No change expected |
| collections | Serves the new navigation pages, browse, topic and services and information pages on GOV.UK | No change expected |
| contacts-admin | Publishes HMRC contact information on GOV.UK | No change expected |
| content-data-admin | A front end for the data warehouse | No change expected |
| content-store | API for content on GOV.UK | No change expected |
| content-tagger | Tool to tag content and manage the taxonomy on GOV.UK | No change expected |
| email-alert-api | Sends email alerts to the public for GOV.UK | No change expected |
| email-alert-frontend | Serves email alert signup pages on GOV.UK | No change expected |
| feedback | Serves contact pages on GOV.UK | No change expected |
| government-frontend | Serves government pages on GOV.UK | No change expected |
| hmrc-manuals-api | API for HMRC to publish manuals to GOV.UK | No change expected |
| imminence | Find My Nearest API and management tools on GOV.UK | No change expected |
| link-checker-api | Checks links on GOV.UK | No change expected |
| maslow | Create and manage needs on GOV.UK | No change expected |
| publishing-api | API to publish content on GOV.UK | No change expected |
| release | Helps deploying to GOV.UK | No change expected |
| router | Router in front on GOV.UK to proxy to backend servers on the single domain | No change expected |
| search-admin | Admin for GOV.UK search | No change expected |
| short-url-manager | Tool to request, approve and create short URL redirects on GOV.UK | No change expected |
| signon | Single sign-on service for GOV.UK | No change expected |
| smart-answers | Serves smart answers on GOV.UK | No change expected |
| support | Forms to raise Zendesk tickets to be used by Government personnel on GOV.UK | No change expected |
| support-api | API for processing GOV.UK named requests and anonymous feedback | No change expected |
| transition | Managing redirects for sites moving to GOV.UK. | No change expected |
| local-links-manager | Manages local links from local authorities on GOV.UK | No change expected |

[princ]: https://docs.google.com/document/d/1Oft4akc6dZfhhOjosNPbFpcLUOUjz7YG7QPcVZi8hww/edit#heading=h.goscl46jcc91
