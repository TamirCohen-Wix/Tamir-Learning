# **Catalog Calendar Guide**

## **Table of Contents**

- [**Introduction**](#introduction)
- [**Terms**](#terms)
- [**Catalog Calendar Projects**](#catalog-calendar-projects)
- [**Usefull links**](#usefull-links)
- [**Issues and who to contact?**](#issues-and-who-to-contact)
- [**GA OOI Component and known issues**](#ga-ooi-component-and-known-issues)
- [**Tech Known Issues**](#tech-known-issues)
- [**Product Known Issues**](#product-known-issues)
- [**APP Reflow**](#app-reflow)
- [**SSR**](#ssr)
- [**Experiments**](#experiments)

## **Introduction**

This document is intended to provide a high level overview of the catalog calendar projects along with monitoring/artifacts and common problems.

## **Terms**

OOI component -
An application with two logical parts:

- Component code (view components) written in React and run in the Viewer
- Controller Code - business logic code that runs inside the platform worker<br>
More details can be found <a href="https://bo.wix.com/wix-docs/fe-guild/viewer/platform/intro/get-started/guidelines">here</a>

OOI is also rendered in SSR (Relevant for landing page only)

## **Catalog Calendar Projects**

### ![#28cb61](https://placehold.co/10x10/28cb61/28cb61.png) <b>Services list widget / Book online page - OOI component</b>

#### Book Online Page (Main bookings page)
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-service-list-widget">bookings-service-list-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12/book-online">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/mQQ-cia7k/auto-fedops-tb-bookings-service-list-page?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/page-out-of-iframe?component-id=621bc837-5943-4c76-a7ce-a0e38185301f">Dev Center</a>
- Artifacts:
  - Viewer (Live site): BookOnlineController.bundle.min.js, BookOnlineViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: BookOnlineController.bundle.min.js, BookOnlineViewerWidget.bundle.min.js
  - Studio Editor: BookOnlineController.bundle.min.js, BookOnlineViewerWidgetNoCss.bundle.min.js
  - Settings panel: BookOnlineSettingsPanel.bundle.js

#### Services List Widget (User can install calendar component on any page of his site)
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-service-list-widget">bookings-service-list-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12/services-list-widget">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/g0NKewAnk/auto-fedops-tb-bookings-service-list-widget?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/widget-out-of-iframe?component-id=cc882051-73c9-41a6-8f90-f6ebc9f10fe1">Dev Center</a>
- Artifacts:
  - Viewer (Live site): ServiceListWidgetController.bundle.min.js, ServiceListWidgetViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: ServiceListWidgetController.bundle.min.js, ServiceListWidgetViewerWidget.bundle.min.js
  - Studio Editor: ServiceListWidgetController.bundle.min.js, ServiceListWidgetViewerWidgetNoCss.bundle.min.js
  - Settings panel: ServiceListWidgetSettingsPanel.bundle.js

#### App settings migration
The standart way to save widget settings in editor is done by yoshi which use editor APIs to save data on public data and styles param.
- Styles params refer to fonts, colors, sizes (px)
- Public data refer to layouts/ which services to show etc..<br>

We had legacy service list settings panel which saved data on app settings (both styles param & public data) and 
in order to align with the guidelines we run lazy migration which migrate app settings to styles param and public data.<br>

The migration steps:
- User open service list settings panel in the editor: 
  * If app settings data exist for this widget (`externalId !== undefined`)
    * get app settings data by external id
    * save all the data on public data and styles param
    * notify user (if migrated data is not same as the app settings data)

#### APIs
<img src="images/ServicesListAPICalls.png" alt = "Services List API calls"/>


### ![#2890e7](https://placehold.co/10x10/2890e7/2890e7.png) <b>Service Page - OOI component</b>

- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-service-details-widget">bookings-service-details-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12/service-page/filters">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/zaGSLE5Mz/auto-fedops-tb-bookings-service-details-widget?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/page-out-of-iframe?component-id=a91a0543-d4bd-4e6b-b315-9410aa27bcde">Dev Center</a>
- <a href="https://bo.wix.com/_serverless/heartbeat/applications/wix-private:scheduler:service-page">Heartbeat</a>
- Artifacts:
  - Viewer (Live site): BookingServicePageController.bundle.min.js, BookingServicePageViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: BookingServicePageController.bundle.min.js, BookingServicePageViewerWidget.bundle.min.js
  - Studio Editor: ServiceListWidgetController.bundle.min.js, BookingServicePageViewerWidgetNoCss.bundle.min.js
  - Settings panel: BookingServicePageSettingsPanel.bundle.js

#### APIs
<img src="images/ServicePageAPICalls.png" alt = "Service Page API calls"/>


### ![#3cd7cc](https://placehold.co/10x10/3cd7cc/3cd7cc.png) <b>Calendar Page / Calendar Widget / Weekly Timetable / Daily agenda - OOI components</b>

#### Calendar Page
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-calendar-widget">bookings-calendar-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12/booking-calendar/filters">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/m_xGJr4Ik/auto-fedops-tb-bookings-calendar-page?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/page-out-of-iframe?component-id=54d912c5-52cb-4657-b8fa-e1a4cda8ed01">Dev Center</a>
- Artifacts:
  - Viewer (Live site): BookingCalendarController.bundle.min.js, BookingCalendarViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: BookingCalendarController.bundle.min.js, BookingCalendarViewerWidget.bundle.min.js
  - Studio Editor: BookingCalendarController.bundle.min.js, BookingCalendarViewerWidgetNoCss.bundle.min.js
  - Settings panel: BookingCalendarSettingsPanel.bundle.js

#### Calendar Widget (User can install calendar component on any page of his site)
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-calendar-widget">bookings-calendar-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12/calendar">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/uMMjgD-Mz/auto-fedops-tb-bookings-calendar-widget?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/widget-out-of-iframe?component-id=0eadb76d-b167-4f19-88d1-496a8207e92b">Dev Center</a>
- Artifacts:
  - Viewer (Live site): BookingCalendarWidgetController.bundle.min.js, BookingCalendarWidgetViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: BookingCalendarWidgetController.bundle.min.js, BookingCalendarWidgetViewerWidget.bundle.min.js
  - Studio Editor: BookingCalendarWidgetController.bundle.min.js, BookingCalendarWidgetViewerWidgetNoCss.bundle.min.js
  - Settings panel: BookingCalendarWidgetSettingsPanel.bundle.js

#### Weekly Timetable (User can install weekly timetable on any page of his site)
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-calendar-widget">bookings-calendar-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/Qih1jgn4z/auto-fedops-tb-bookings-weekly-timetable-widget"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/widget-out-of-iframe?component-id=3c675d25-41c7-437e-b13d-d0f99328e347">Dev Center</a>
- Artifacts:
  - Viewer (Live site): WeeklyTimetableController.bundle.min.js, WeeklyTimetableViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: WeeklyTimetableController.bundle.min.js, WeeklyTimetableViewerWidget.bundle.min.js
  - Studio Editor: WeeklyTimetableController.bundle.min.js, WeeklyTimetableViewerWidgetNoCss.bundle.min.js
  - Settings panel: WeeklyTimetableSettingsPanel.bundle.js

#### Daily Agenda (User can install daily agenda on any page of his site)
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-calendar-widget">bookings-calendar-widget</a>
- <a href="https://noak56.wixsite.com/my-site-12/daily-agenda">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/MRaMJrVSz/auto-fedops-tb-bookings-daily-agenda-widget?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/widget-out-of-iframe?component-id=e86ab26e-a14f-46d1-9d74-7243b686923b">Dev Center</a>
- Artifacts:
  - Viewer (Live site): DailyAgendaController.bundle.min.js, DailyAgendaViewerWidgetNoCss.bundle.min.js
  - Classic Editor / Editor x: DailyAgendaController.bundle.min.js, DailyAgendaViewerWidget.bundle.min.js
  - Studio Editor: DailyAgendaController.bundle.min.js, DailyAgendaViewerWidgetNoCss.bundle.min.js
  - Settings panel: DailyAgendaSettingsPanel.bundle.js
  
#### APIs
<img src="images/CalendarPageAPICalls.png" alt = "Calendar Page calls"/>

### ![#cdc5fa](https://placehold.co/10x10/cdc5fa/cdc5fa.png) <b>Bookings Viewer Script</b>
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-viewer-script">bookings-viewer-script</a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/platform?component-id=85fa4310-61df-4e51-8f17-7ecfeccc126d">Dev Center</a>
- Artifacts:
  - Viewer script: bookingsViewerScript.bundle.min.js - This artifact is application script that is downloaded one time once page with Bookings widget is loaded. <br>
  This script has two functions that are triggered by the viewer
  ```js
  initAppForPage : function(initParams, platformApis, scopedSdkApis, platformServicesApis){
    //initialization code
   },
   createControllers: function(configs){
    //return controller by component id
   }
   ```
  - Editor script: bookingsEditorScript.bundle.min.js - This artifact is application script that is downloaded in editor and responsible for post installation functionalities/ app manifest and more

### ![#6a5acd](https://placehold.co/10x10/6a5acd/6a5acd.png) <b>Frontend server that server Book online page & Services list widget</b>
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-widget">bookings-widget</a>
- <a href="https://fryingpan.wixpress.com/services/com.wixpress.bookings-widget">fryingpan</a>

#### Fes Endpoint
<img src="images/bookingsWidget.png" alt = "Bookings widget endpoint"/>

### ![#cd2667](https://placehold.co/10x10/cd2667/cd2667.png) <b>Single Service Widget - Legacy</b>
Users cannot install this widget anymore but there are existing widgets on sites that were installed before we removed this widget from the add panel.<br>
User can install new single service widget with dedicated preset of service list widget.

- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/bookings-widget-viewer">bookings-widget-viewer</a>
- <a href="https://noak56.wixsite.com/mysite-12/service-widget">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/hXz4mZpGk/auto-fedops-tb-bookings-single-service-widget?orgId=1"> Fedops </a>
- <a href="https://dev.wix.com/apps/13d21c63-b5ec-5912-8397-c3a5ddb27a97/extensions/dynamic/widget-out-of-iframe?component-id=14756c3d-f10a-45fc-4df1-808f22aabe80">Dev Center</a>
- Artifacts:
  - Viewer (Live site): bookingsViewerScript.bundle.min.js (old non yoshi controllers are bundled to main viewer script), component.bundle.min.js
  - Classic Editor / Editor x: app.bundle.min.js - controller and component are bundled together (not rendered as ooi in editor)
  - Settings panel: https://static.parastorage.com/services/scheduler-widget/1.809.0/scripts/modules-settings.js (Angular scheduler-widget)

### ![#f6b09e](https://placehold.co/10x10/f6b09e/f6b09e.png) <b>Bookings Services Preferences Modal </b>

Allow user multiple selections for multiple services (Beauty flow)
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/modules/bookings-services-preferences-modal">bookings-services-preferences-modal</a>
- <a href="https://noak56.wixstudio.io/beauty/treatments">Example - click on Book on one of the services</a>

#### APIs
<img src="images/MultiplePreferencesAPICalls.png" alt = "Calendar Page calls"/>

### ![#282f61](https://placehold.co/10x10/282f61/282f61.png) <b>Bookings App Builder Controllers</b>
- App builder widgets (Old Blocks) - Old daily agenda, staff widget, book button<br>
  The react components are built with editor components.<br>
  This module export controller for each widget mentioned above and those controllers are bundled to bookingsViewerScript.bundle.js

#### Daily agenda - legacy
Users cannot install this widget anymore but there are existing widgets on sites that were installed before we removed this widget from the add panel<br>
User can install new daily agenda widget (Calendar section)

- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/modules/bookings-app-builder-controllers">bookings-app-builder-controllers</a>
- <a href="https://noak56.wixsite.com/my-site-12/blank-4">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/GYmqwP5Gz/auto-fedops-tb-bookings-daily-timetable?orgId=1"> Fedops </a>

#### Staff widget
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/modules/bookings-app-builder-controllers">bookings-app-builder-controllers</a>
- <a href="https://noak56.wixsite.com/my-site-12/staff">Example</a>
- <a href="https://fedops-grafana.wixpress.com/d/RQz3QE5Gk/auto-fedops-tb-bookings-staff-widget?orgId=1"> Fedops </a>

#### Book button
- <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/tree/master/packages/modules/bookings-app-builder-controllers">bookings-app-builder-controllers</a>
- <a href="https://noak56.wixsite.com/my-site-12">Example</a>


## **Usefull links**

- <a href="https://bo.wix.com/wix-docs/fe-guild/editor-platform/developer-guides">Editor Platform</a>
- <a href="https://bo.wix.com/wix-docs/fe-guild/viewer">Viewer Platform</a>
- <a href="https://github.com/wix-private/wix-design-systems/tree/master/packages/wix-ui-tpa">Wix UI TPA</a>
- <a href="https://tableau.wixpress.com/#/site/AllWix/views/VerticalsScores/VerticalsScores/654ddf53-3baf-46cd-8bc1-164433bfe53d/bb7a974b-e621-4b22-8a64-d49937deaeb9?:display_count=n&:showVizHome=n&:origin=viz_share_link">Performance (Choose Bookings)</a>
- <a href="https://www.wixlightkeeper.com/">Create own benchmarks</a>
- <a href="https://bo.wix.com/wix-docs/rnd/getting-started">Wix Docs Internal</a>
- <a href="https://dev.wix.com/docs">Wix Docs</a>
- <a href="https://www.wix.com/velo/reference/">Velo</a>
- <a href="https://bo.wix.com/ci-police-station/projects?ownershipTag=bookings-uou">CI Police bookings-uou</a>
- <a href="https://bo.wix.com/_serverless/app-service-autorelease/manage">Auto Release Manager</a>
  - Once GA is done then version are being updated in dev center according to artifact configuration
- <a href="https://bo.wix.com/dev/artifacts">Dev Portal</a> 
  - GA/ Deploy preview
- <a href="https://pbo.wix.com/fire-console/">Fire console</a>
- <a href="https://bo.wix.com/dumbledore">Dumbledore</a> 
  - Bundles size/ Dependency tree
- <a href="https://bo.wix.com/bi-catalog-webapp/#/sources/16/events/564?artifactId=wix.boost.artifact.id/">Bookings BI Catalog</a>
- <a href="https://bo.wix.com/bi-ux/#/realtime?levels=1&fields=date_created,src,evid,evid_desc,browser_name,app_url&domain=user&uuid=89ad6dfb-cff0-483d-9350-30699472c1a1">WIX BI Explore</a>

## **Issues and who to contact?**

- **ssr alerts - You can try yourself if there is a general issue in SSR**
    - https://noak56.wixsite.com/my-site-12/book-online?ssrOnly
    - https://noak56.wixsite.com/my-site-12/service-page/filters?ssrOnly
    - If the page is rendered as expected so you can try and write in <a href="https://wix.slack.com/archives/CL5TBPB8S">thunderbolt channel</a>
- **widgets are not rendered in SSR**
  - Take the request id of the html
  - Put it in grafana - <a href="https://grafana.wixpress.com/d/38cCoLymz/error-analytics-traceid?orgId=1&var-app=1&var-request_id=1609943103.988263048916829767&var-filter=1">example</a> 
  - Check if you see any error related to bookings servers, if you do you can write to the relevant group in Bookings, if not so you can start with <a href="https://wix.slack.com/archives/CL5TBPB8S">thunderbolt</a>
- **widget is not loaded on live site**
  - The error you see on network seems like general error that not related to bookings code
  > Error: Mismatched anonymous define() module: [object Object]
  https://requirejs.org/docs/errors.html#mismatch
  at makeError (requirejs.min.js:1:1795)
  at T (requirejs.min.js:1:8677)
  at Object.s [as require] (requirejs.min.js:1:15042)
  at requirejs (requirejs.min.js:1:2335)
  at componentsLoaderClient.ts:71:5
  instrument.ts:124 Error: Mismatched anonymous define() module: [object Object]
  https://requirejs.org/docs/errors.html#mismatch
  at makeError (requirejs.min.js:1:1795)
  at T (requirejs.min.js:1:8677)
  at requirejs.min.js:1:15068
  - Users might have custom script that cause the viewer to crash
  - You can try to add to url: `?disableHtmlEmbeds=true`
  - If that fix the issue then you need to understand which script cause it - maybe disable one by one in network and once you find it you can notify the user about it

## **GA OOI Component and known issues**
  - OOI artifacts GA is controlled by DAC (Relevant for viewer - live site)
  - Wix employees will get latest RC version in live site
  - Once you GA:
    - Live site will get the artifact version by DAC, the last version will be gaed gradually.
    - Editor will get GA version immediately
      - Each OOI component has <a href="https://bo.wix.com/_serverless/app-service-autorelease/manage">auto release</a> configuration that update the dev center artifacts once you ga.
      - Once the dev center artifact is updated the Client spec map is updated and this way the editor knows which bundle and versions to load for each component.
  - **When you ga and u dont get expected version:**
    * In live site - write in <a href="https://wix.slack.com/archives/C0193H2TS48">dac support</a> 
    * In editor - check the dev center version and if it's not updated with ga version so write to <a href="https://wix.slack.com/archives/C4S33T37H">dev center</a>

## **Tech Known Issues**

- css per bp not working in local env
- In live site the css is rendered in thunderbolt (site assets server), this is why the bundle in live site has `NoCSS` suffix.
- In Editor the css is rendered by the component (The main component is wrapped with `withStyles` component), the component bundle in the editor is different than the live site and it doesnt have the `NoCss` suffix.

## **Product Known Issues**

- **Non repeated session** 
  - When owner define non-repeated session for specific service in the back office calendar and assign to it staff which is not part of any recurring session:
    - This staff won`t be shown as part of staff filter in Calendar / Weekly timetable/ Daily agenda component. 
    - When selecting this staff in staff widget and navigating to assigned services list we wont get this service because the staff is not part of the recurring data thus he is not part of the catalog data
- **Add link to editor button**
  - This feature is not Bookings feature, its editor platform feature which use Bookings site map endpoint to get urls to our services page / calendar page.

## **APP Reflow**

- Velo users can replace service page and calendar page with their own custom pages
  - <a href="https://dev.wix.com/docs/develop-websites/articles/wix-apps/wix-bookings/build-a-custom-booking-calendar-page">Docs</a>
  - Once user show an intent through the editor to replace the page then the editor is replacing the static page with dynamic page and add router to that page 
  - The viewer trigger the router endpoint once navigating to page with router:
    - `_serverless/bookings-viewer-router/pages` 
    - The router response contains the service info
    - <a href="https://github.com/wix-private/bookings-calendar-catalog-viewer/blob/master/serverless/bookings-viewer-router/src/index.ts">Code</a>

## **SSR**

- Render SSR only: https://noak56.wixsite.com/my-site-12/book-online?ssrOnly
- We dont render calendar data in SSR (Sections that contain this data are lazy loaded in CSR)
  * Course availability info in Service list page/widget
  * Sessions info in Service page
  * Slots and date picker in Calendar page/widgets 

## **Experiments**

- Conduct in OOI component
  * `viewer-apps-13d21c63-b5ec-5912-8397-c3a5ddb27a97`
  * `viewer-apps-owner-scope-13d21c63-b5ec-5912-8397-c3a5ddb27a97` - Conduct by owner - All owner sites will get same value
- You cannot open experiment by language (Such conduction not exist in centralize conduction in viewer server) 
  - When you open an experiment in live site you need to make sure all new keys are translated
- If you add new feature which is based on new data that was added in the Back office then u need to wrap your changes in live site by FT and make sure this FT is gradually rolled-out before opening the BO feature. (AKA - Feature Killer)




