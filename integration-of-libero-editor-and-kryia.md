# Integration of Libero Editor and Kryia

## Introduction

The document provides a high level overview of how Libero Editor will integrate with Kriya. Its purpose is to capture the decisions of various technical discussions into a high level architecture document that details how the various components of the system will interact, and what each part of the system is responsible for.

## High Level Architecture

The below diagram provides a high level overview of the proposed solution.

[![](.gitbook/assets/0%20%281%29.png)](https://www.draw.io/?page-id=gtG6yaVi1xr8IQmn1TSe&scale=auto#G1W_NgMkl79-qhfj9nuLOKEO9glxRT5iQE)

The components above are detailed below. Note that examples are just for reference only, final message structures, etc, need to be fleshed out. Also it is assumed that API calls are authenticated, and done over HTTPS.

### Kriya 1 Workflow and CMS

Handles converting an article from its submission format - usually a word document although approx 10% can be LaTex - into an XML document. Once the pre-editing phase is complete, the article should be written to an S3 bucket so that it can be imported into eLife’s Backend.

### eLife Backend

This is a backend that eLife is building for importing an article and storing it whilst it goes through the QC process. The backend itself is a suite of microservices that together allow a user via the Editor to login, fetch, edit and save an article. The backend will monitor an S3 bucket looking for new articles, and when a new article has been deposited by Kriya import it into its own internal storage.

The backend needs to be able to send updates about articles to Kriya, so that the dashboard can still be used to display the current status of an article. This could be done via a REST API, which allows the eLife backend to POST/PUT status information about an article to Kriya, such as when an article enters the ‘post-author’ phase. See [Kriya 2.0 Dashboard]().

In addition, the eLife backend will interact with Exeter’s PDF generation tool, where it can POST/PUT an article to the generator and get a PDF back. eLife’s backend will be responsible for converting any XML sent to the generator to the required format, as well as adding any additional elements/attributes to the XML that are required to complete the conversion. See [Exeter’s PDF Generator]().

Once the QC process has been completed, eLife’s backend will publish the completed article to Continuum by publishing it to an S3 bucket.

### Libero Editor

The editor is a browser based editing tool for XML that interacts with the eLife backend to allow a user to open and edit an article. The editor will interact only with the eLife backend, and have no direct interaction with any of the Exeter services.

The editor itself will be based upon ProseMirror, which will be modified and extended to support eLife’s preferred format of XML.

For a user, the workflow with the editor will be something like the following...

* The editor can be opened in a browser by navigating to a URL, e.g. [https://editor.elifesciences.org](https://editor.elifesciences.org/).
* A user logs into the editor, which interfaces with the eLife Backend to authorise the users credentials.
* The user uses the menus in the editor to view a list or articles, and selects one to open.
* The user makes some changes to the article, e.g. correcting some spelling mistakes.
* The user saves their changes to the article, which are saved to the eLife backend as a set of proposed changes.
* A member of staff opens the article in the editor, and views the proposed changes made by the above user and decides to accept or reject them. Rejected changes are discarded, accepted changes are merged into the master copy of the manuscript.
* When an article is ready to be published, a member of staff uses the editor to update the state of the article to ‘ready-to-publish’, at which point the eLife Backend publishes the article to Continuum via an S3 bucket.

**Note:** In the case of guest users \(e.g. authors\), they can be sent links that include a pre-generated authentication token so they can open a specific article to review and edit without needing to register an account.

### Kriya 2.0 Dashboard

Dashboard used by the production staff at eLife to see the current status of all articles working their way through the QC/production process. eLife want to continue to make use of the Dashboard, and as such would like a means to send updates about article state, etc, so that they are reflected in the dashboard. For example, updating the dashboard when a digest has been added or when the article has moved to the next stage in the workflow.

This could work as follows...

1. Author makes a change in Libero Editor, and confirms that they have finished making changes.
2. The eLife backend notes that the author has finished their checks, and changes the state of the article to ‘post-author-review’.
3. The eLife backend sends a POST message to Kriya with information on the article, and the new state, e.g..

 `POST /api/updateArticleStatus HTTP/1.1`

 `Host: elife2.kriyadocs.com`

 `Content-Type: application/json`

 `{`

 `articleId: “123456789”,`

 `state: “post-author”,`

`}`

1. Kriya updates its internal state, which is then reflected in the dashboard.

### Exeter’s PDF Generator

Service that can be used to generate a typeset PDF from XML. This will expose an API that can be accessed over HTTPS that will allow eLife to POST an article and supporting materials as an archive and after a short wait return a PDF.

This could work as follows...

1. The Editor sends a message to the eLife backend to get a PDF of the article.
2. The eLife Backend gets the article, performs a conversion on the XML to add the required elements/attributes to the XML needed by the PDF generator.
3. The eLife backend sends a POST message to PDF Generator API which includes in the body a binary stream of an archive containing the article and supporting materials such as figures. e.g.

`POST /api/generatePDF HTTP/1.1`

`Host: elife2.kriyadocs.com`

`Accept: application/pdf`

`Content-Type: application/octet-stream`

1. The PDF generator takes the uploaded article and generates a PDF, which it returns to the eLife Backend.
2. The eLife backend forwards the PDF to the editor, which opens a new browser tab to display it to the user.

**Note:** An alternative to the above was to achieve the same using 2 API calls, one to upload the article and the second to request a PDF of that article. If that was the case, it would work much as the above except the initial call should return an ID or similar once the article has been uploaded, and the second call should use that ID either in the URL or body when requesting a PDF.

## Questions

Place to capture questions and answers about the above.

* How will PDF generation be handled?
  * Exeter: When the request to PDF generation is sent, there would be an API request that is sent and the PDF is generated by Kriya. Need to discuss how the data has to be translated from the Editor data format to Kriya data format.
* Table setter - Needs to be discussed. In Kriya, we have a separate service to handle Table setting. Need to understand how Editor will be handling tables.
* Equation - Similar to tables above, need to understand how this will work.
* Digests/Decision letters - Need to discuss the workflow as content will be updated during the workflow in both the CMS and the editor.
  * Joel: Thoughts are that these will now be uploaded into the eLife Backend, but as we would like to use the Kriya Dashboard then there will need to be some sort of API call from our backend to Kriya to let you know that for example a digest has been added
* Need to understand more details about the DAR format and how it is used.
  * Joel: This is quite important. Perhaps not DAR specifically but we do need to clarify what the format for an article will be for exchange between the eLife backend and Kriya. I assume an archive with a manifest, article xml and supporting materials.
* Need to think about how authentication is handled between Kriya and Editor, e.g. to ensure that when a user tries to save changes back to Kriya that they are properly authorised.
  * Joel: With the above proposed design. I don’t think this matters anymore. I think that there needs to be authentication when our backend calls any exeter service, but assume that we could handle this using an API key.

