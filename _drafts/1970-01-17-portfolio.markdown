---
layout: post
title: A small portfolio of current work
---
I have assembled some screen shots from some of the work I have done at my current job as a demonstration of the attention to detail that goes into applications I work on. Each screenshot has a little story and a lot of code powering it. These are large desktop class applications delivered cross platform in IE9, Firefox, and Chrome (and Safari and Opera as well). 

<i>Please keep in mind that the applications shown below are Copyright Fair Isaac Corporation. Unfortunately I am unable to host live versions of these apps because of corporate policies.</i>

# Insurance Fraud Manager
This application is used by some very large health insurance companies to detect, manage, investigation, and track fraud. The goals for this app from a development perspective are:

1. Secure - Patient health data needs to be protected at all times.
2. Fast - Quick navigation and data loading will lead to increased productivity.
3. Intuitive - The workers using the app will not be experts, and need to be able to understand what they're doing without explicit instruction.
4. Robust - The app needs to recover in the face of errors either client or server side.

<hr>
Claim review screen - Notice the condensed data with many avenues of interaction, from expandable table rows to dynamically clickable procedures and modifiers to reveal underlying codes and descriptions.
![Claim Review Screen](/content/images/2015/03/Claim-Review.png)

<hr>
Investigation Summary - Again, a lot of data is displayed in a succinct way. Individual sections are loaded just-in-time, to keep the page load time down.
![Investigation Summary](/content/images/2015/03/Investigation-Summary.png)

<hr>
Edit Dialog - This modal dialog brings attention to the task at hand without removing the user's work context. The modified bootstrap datepicker is clean and non-intrusive. The dates are marked as invalid by red coloring.
![](/content/images/2015/03/Edit-Activity-With-Datepicker.png)

<hr>
User Assignment - A common task is to assign items to parent items. To do this, we present a left-right comparison with a typeahead multiselect on the left and a simple editable list on the right. Already selected elements are grayed out.
![](/content/images/2015/03/User-Assignment-Left-Right.png)

<hr>
Quick Search Bar - Ever-present in the app is a quick search bar that can be used to search over any facet of data pertinent to the application. This bar is called up with a single click and dismissed as easily.
![](/content/images/2015/03/Quick-Search-Bar.png)

<hr>
Smooth line graph - In IFM, provider data can be explored. We show a few different graphs, one of which is a smoothed line graph with a single point representing the current provider. The graph below is entirely metadata-driven, powered by a large backend REST API. This means all the points, labels, colors, etc. can be changed in the backend and applied in general to the front-end application. It may seem like a simple chart, but in fact required thousands of lines of code to make fully generic.
![](/content/images/2015/03/Smoothed-Line-Graph.png)

Box and whisker plot - Again, this chart is metadata driven.  This one shows quartiles for each group.
![](/content/images/2015/03/Box-and-Whisker.png)

# Dashboard Framework
Dashboard framework is an app build on top of Kibana 3. It is meant to talk to Elasticsearch and display data in many useful ways. Dashboard is meant to be exploratory; quickly diving into and finding meaning in very large data sets that have been pre-indexed for speed. For FICO, we added two panels to Kibana that add unique text analytics capabilities.

Goals

1. Support all built in Kibana panels and charts by extending existing data models and creating a custom theme.
2. Add 2 panels and several overlays for interacting with text specific data.
3. Add a security model to handle multiple users / tenants with SSO.
4. Make Kibana completely embeddable to be used in other products, exposing an advanced runtime API to interact with all application elements programmatically.
5. Speed, speed, speed. Data exporation must be fast and efficient.

<hr>
Dashboard Framework - An overview of the app. Note that this is a completely re-skinned version of Kibana 3.
![](/content/images/2015/03/DF---Overview.png)

<hr>
Associated terms network graph - When viewing terms, a user can dive into one to view statistically associated terms in a small graph. This panel also summarizes the analytic data.
![](/content/images/2015/03/Associated-Terms-Graph.png)

<hr>
Network Graph - This graph shows more complicated relationships between "targets." Each target has associated terms. Terms inside the larger circle are shared between 2 or more targets. Terms outside are unique to their respective targets. 

This graph is built on D3 using the force layout. It can be zoomed, panned, and interacted with to identify meaningful relationships between targets.
![](/content/images/2015/03/Network-graph.png)

<hr>
Chord Viewer - A compliment to the network graph, the chord viewer uses D3's chord layout to show the number of shared terms between targets in the thickness of the bands. Cooler band colors represent less shared terms, hotter colors represent more shared terms. The diagram is interactive, with tooltips showing a list of top shared terms. Targets can be faded out by clicking on them to highlight all chords in and out of a single target (this is better demoed live).
![](/content/images/2015/03/Chord-Diagram.png)

<hr>
Document Annotation - This view shows sentiment analysis for a single document. By clicking on a sentiment on the left, the words predicting that sentiment are highlighted, while the surrounding words are blurred. 
![](/content/images/2015/03/Document-Annotation.png)

<hr>
Document Search: Facet Selection - On this panel the user can search over facets, like searching for Amazon products, and see their searches applied in real time. The labels at the top represent chosen facets. The user has the option to apply custom names and colors to the labels.
![](/content/images/2015/03/Facet-Selection.png)

<hr>
Term cloud view - For a better bird's eye view of important terms in a document, a cloud view can be used. This uses D3 and some open source software to display.
![](/content/images/2015/03/Cloud-View.png)
