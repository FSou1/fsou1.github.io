---
title: Customer analysis solution architecture with Azure
tags: .NET Azure Architecture
redirect_from: "/Customers_analysis_with_azure/"
---

I have recently been asked to design a solution architecture to analyze store customers. The key feature of the solution is to recognize human emotions and provide reports based on collected data.

## Description

A client has a retail business with lots of shops distributed all over the world. Checkouts are equipped with cameras that shoot customers once they've paid and headed for the exit through security gates. The following data should be collected, analyzed, and used in reports to client managers:

- Gender;
- Age;
- Mood (emotion).

A few more requirements that have to be considered:

- Number of stores: 1000 (around the world);
- Working hours: 8AM - 8PM;
- Average number of customers per store: 5000 people a day;
- Number of cameras per store: 10;
- Network throughput: 50 Mbps;
- Retention period: 1 year.

The client expects the following result:

- Camera requirements (to ensure the current ones can be reused);
- A reporting tool that is available via Web UI and provides reports and diagrams by region, city, and specific store;
- An Azure-based solution architecture.

## Solution

Before designing a solution, we have to calculate amount of data being collected and analyzed on a regular basis. First of all, we have to realize that because of different time zones, there are just around 500 active stores every hour. Secondly, each of the store serves 5000 customers a day or about 415 customers per hour, which means there are:

- 415 \* 500 = 207,500 customers per hour in the world;
- 207,500 / 60 = 3,450 customers per minute;
- 207,500 / 3600 = 60 customers per second.

In other words, a solution architecture should be able to handle at least 60 requests per second on a regular working day. It should also be scalable enough to address not only increased workload on holidays and weekends, but further business growth as well.

In terms of data volume, it means that once we've decided to shoot each customer with a 1MB photo, eventually we'll receive:

- 1,000 \* 5,000 = 5,000,000 photos a day;
- 5,000,000 \* 1MB = 5 TB of data a day;
- 5 \* 30 = 150 TB a month.

Fortunately, inbound data transfers (i.e. data going into Azure data center) are free (nevertheless, a price for storing such amount of data is still a relevant concern).

Before this project, I've never faced a task to provide camera requirements. So I decided not to limit them too much and focus mostly on the design. A few of them are below:

- Ability to set up working hours (staff should not be shot and analyzed while stores are closed);
- Ability to shoot by a trigger (e.g. PIR sensor);
- P2P support (this technology enables to connect to and control devices without using static IP addresses and uniquely identify them by UID);
- FTP support (a camera should be able to upload images or videos to a remote FTP server);
- Ability to uniquely identify a camera based on uploaded photo or video (e.g. geo-tagging, file name setting or specific camera ID in file metadata). This information will make it possible to provide reports and diagrams by regions, cities, and even specific stores;
- Face size in the photo or video must be at least 200x200 pixels (and 100 pixels between eyes).

Now that all the requirements are defined, I believe we can dig into details of the proposed design.

![customer-analysis-with-azure](/assets/images/posts/customer-analysis-with-azure.png)

Once a customer came through a security gate, a camera shoots him and uploads the photo to an FTP server. The request will be routed by [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) to a relevant FTP server based on a region. The service provides DNS-based traffic load balancing and enables us to distribute traffic optimally to services across global Azure regions, while providing high availability and responsiveness.

Since Azure Blob Storage doesn't support the ftp protocol, we have to configure it on our own. To do that we need to deploy a load-balanced, high availability FTP Server with Azure Files ([read more](http://fabriccontroller.net/deploying-a-load-balanced-high-available-ftp-server-with-azure-files/)).

In order to increase the availability and reliability of the VMs, they can be combined into an [Availability Set](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/tutorial-availability-sets). Azure makes sure that the VMs you place within an availability set run through multiple physical servers, compute racks, storage units, and network switches. If a hardware or software failure happens, only a subset of your VMs is impacted and your overall solution stays operational.

Let's assume that a photo has been uploaded to an FTP server. After that we have to process the photo and persist required information in the storage. [Azure Logic App](https://docs.microsoft.com/en-us/azure/logic-apps/) is a good choice for this scenario. Every logic app workflow starts with a trigger, which fires when a specific event happens, or when new available data meets specific criteria. In our case, an [FTP trigger](https://docs.microsoft.com/en-us/azure/connectors/connectors-create-api-ftp) is the one we're going to use (keep in mind that FTP actions support only files that are 50 MB or smaller). Once the trigger is fired, we have full control over the processing workflow (e.g. call a service and persist data in a storage).

In order to collect required information, such as age, gender, and mood, I would suggest using [Azure Face API](https://docs.microsoft.com/en-us/azure/cognitive-services/face/overview) of Cognitive Services. The API can detect human faces on an image and extract a series of face-related attributes such as pose, head pose, gender, age, emotion, facial hair, and glasses. We can even find similar faces or build a database of customers, but for now it's out of the scope.

> Face API requires that image file size should be no larger than 4 MB.

Truth be told, running costs for 150,000,000 requests in comparison with the remaining services are quite noticeable: about $90,000 a month. In case it's the primary concern, then the following options have to be considered:

- Reduce a number of API requests by combining multiple photos in a single one;
- Integrate with other face recognition vendors (e.g. [Face++](https://www.faceplusplus.com/v2/pricing-details/#api_1));
- Use available deep learning models and open-source libraries (e.g. [OpenCV](https://docs.opencv.org/2.4.13.7/modules/core/doc/intro.html), [Dlib](http://blog.dlib.net/), [OpenFace](https://cmusatyalab.github.io/openface/)).

I believe the rest of the design isn't surprising. Once face information has been extracted, it can be easily persisted to any storage in a structured format (such as JSON) and retrieved by [Power BI](https://docs.microsoft.com/en-us/power-bi/) for further analysis.

## Summary

Is it the only valid design for the provided requirements? Definitely not. Microsoft and Azure provides a huge number of first-class services, with the choice of often depending on many factors, such as quality attributes, constraints, running costs, trade-offs, and production readiness. When making the choice, experience is definitely not the last factor.

> We are all products of our experience and it influences architecture and architectural decisions.

Thank you for reading and any feedback is welcome!

Reference:

1. [Exploring Azure Logic Apps with an IP Camera](https://geoffhudik.com/tech/2018/01/15/exploring-azure-logic-apps-with-an-ip-camera/);
2. [Azure Face-API And Logic App (Serverless)](https://www.azureitis.nl/azure-face-api-and-logic-app-serverless/#comment-1844);
3. [Build a Cloud-Integrated Surveillance System Using Microsoft Azure ](https://www.petri.com/build-cloud-integrated-surveillance-system-using-microsoft-azure-windows-10-part-1);
4. [Deploying a load-balanced, high-available FTP Server with Azure Files](http://fabriccontroller.net/deploying-a-load-balanced-high-available-ftp-server-with-azure-files/);
5. [FTP server proxy for Azure Blob Storage](http://www.redbaronofazure.com/?p=5781);
6. [Azure Functions + Cognitive Services: Automate Image Moderation](https://techbrij.com/azure-functions-cognitive-services-automate-moderation);
7. [Face Analysis Camera Selector](https://www.acti.com/faceanalysisselector).
