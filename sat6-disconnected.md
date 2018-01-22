## Overview

This guide provides guidance as to how a user of Red Hat Satellite 6 can setup a Satellite installation to support a disconnected.


## Intended Audience

You love Satellite and all things Satellite.

## Source Material / Additonal Reading

This document is meant as a supplement to the existing product docs. Content for this guide was sourced from (and inspired by)

- The Arch guide
- The install guide
- The content management guide
- The ISS Knowledge article
- Subscription Manager for the Recovering RHN Addict, part 7

## What is disconnected?


In the scope of Red Hat Satellite, **disconnected** refers to any Satellite which cannot reach **cdn.redhat.com**. Many Satellites are built disconnected because of a number of reasons, including, but not limited to

* Organizational policies which do not allow production systems to connect directly to the internet
* Completely Airgapped systems which cannot connect to the internet at all.


This guide will cover how to

* Plan your disconnected environment from an architectural and subscription perspective
* Installing your first disconnected environment
* Importing content for the first time
* Maintaining the disconnected environment and its content over time.

This document will be written 'case-study' style and is intended to be a cookbook.


## Planning

There are two major mechanisms that can be used to acquire Red Hat content, **Content ISOs** or **Red Hat Satellite Inter-Satellite-Sync**

Content ISOs can be downloaded from the REd Hat Customer Portal (Insert Link). Content ISOs are periodic snapshots of a product's repositories, made on a scheduled basis (usually every quarter give or take). Content ISOs can be downloaded from the customer portal, extracted and used to populate a disconnected Satellite.

(PUT DIAGRAM HERE)

Alternatively, a user can deploy a Satellite which *is* able to connect to cdn.redhat.com and use that Satellite to export Content suitable for importing into another Satellite which is disconnected.  

(PUT DIAGRAM HERE)

There are pros & cons to each approach


Method | Pros | Cons
-------- | -------- | --------
Inter Satellite Sync | Can export any/all content which you have a valid subscription for. | Cost (both hardware & subscription) for the internet connected Satellite.
Content ISOs | Do not require an Internet connected Satellite. | Aren't released often. Aren't available for all products.


It is *strongly* recommended to use **Inter Satellite Sync** as it puts the total control over when content is exported/imported in the hands of the user. Content import / export using content ISOs will be covered in an Appendix to this doc.

### Planning Subscriptions & Manifests.

In our first case study, Darrell Disconnected, a systems administrator at Example Corp, wishes to support 2 Satellite servers, one which will be used to support systems in the disonnected environment, and  one which will be internet connected with the intention of using it to export connect. Additionally, Darrell needs to manage 100 Red Hat virtual machines in the disconnected environment.

**Which subscriptions to purchase?**

In this example, Darrell needs to purchase two Satellite server subscriptions (MCT0370), one for each of the connected and disconnected environments respectively. Note: Red Hat Satellite Starter Pack subscriptions (MCT1650) can (and should) be used if the number of managed RHEL systems in each environment is less than 50.

Darrell's account has the following subs:


Subscription | Quantity | Purpose
-------- | -------- | --------
MCT1650 - Red Hat Satellite Starter Pack | 1| Internet Connected Satellite for exporting content.
MCT0370 - Red Hat Satellite | 1 | Disconnected Satellite
RH00008 - Red Hat Enterprise Linux with Smart Management (Physical or VirtualNodes) | 50 | RHEL + Smart Management Subscriptions for the managed Nodes. 

Note: THe RH00008 Subscriptions are example, and indicative of the deploy. Your subscriptions will vary. (It is only the Satellite subscriptions that matter.)


