## Overview

This guide provides guidance as to how a user of Red Hat Satellite 6 can setup a Satellite installation to support a disconnected.

## Version Support

This document has been developed on Satellite 6.3 (Beta). It _should_ work with Satellite 6.2 with minor tweaks.

## Intended Audience

You love Satellite and all things Satellite.

## Source Material / Additonal Reading

This document is meant as a supplement to the existing product docs. Content for this guide was sourced from (and inspired by)

- The Arch guide
- The install guide
- The content management guide
- The ISS Knowledge article
- Subscription Manager for the Recovering RHN Addict, part 7
- The Hammer CLI Guide

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

![alt text](./images/Sat6_Disconnected_with_Content_ISOs.png "Content ISO download")

In this scenario, the user would

1. download content ISOs from the customer portal (insert Link) to a workstation.
2. burn the content ISOs to CD or DVD.
3. transit those ISOs into their disconnected environment and import it into a Satellite

Alternatively, a user can deploy a Satellite which *is* able to connect to cdn.redhat.com and use that Satellite to export Content suitable for importing into another Satellite which is disconnected.  

![alt text](./images/Sat6_Disconnected_with_Connected_Satellite.png "Content ISO download")

In this scenario, the user would
1. synchronize products for which they have a valid subscription for to their internet connected satellite.
2. Export that content using the Inter-Satellite-Sync Feature
3. transit that content to the disconnected Satellite (by either exporting to CD/DVD ISO OR by a disk export)

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

Note: As the internet connected Satellite is only being used to sync/export content, Darell opted to get a Satellite Starter Pack, which is lower cost. Also, the RH00008 Subscriptions are example, and indicative of the deploy. Your subscriptions will vary. (It is only the Satellite subscriptions that matter.)



### Changes to the standard installation.

Satellites which will be used to export content have different disk space & partitioning requirements than usual.  Of note are the `/var/cache/pulp` and `/var/lib/pulp/katello-export` directories.

The `/var/cache/pulp` directory is used as transient space for both importing **AND** exporting content. It needs to be sized as large as the largest content export. If exporting content to ISO, this needs to be sized **twice** as large as the largest content export.

Example: If a repository with 40GB in content needs to be exported, `/var/cache/pulp` needs to be 40GB in size. If this same repository is exported to ISO, 80GB is required in `/var/cache/pulp`

The `/var/lib/pulp/katello-export` directory is the final storage location for all exported content. This directory needs to be sized as large as the largest content export. Example: If a repository with 40GB in content needs to be exported, `/var/lib/pulp/katello-export` needs to be 40GB in size.

 This directory can be changed if necessary:

- In the UI, changing the pulp_export_destination option, under Administer->Settings->Katello.
- via Hammer `hammer settings set --name pulp_export_destination --value /var/www/html/pub/exports/``



### Creating Subscription Manifests.

In our example above, Darrell Disconnected has the following subscriptions associated with his account.

Subscription | Quantity | Purpose
-------- | -------- | --------
MCT1650 - Red Hat Satellite Starter Pack | 1| Internet Connected Satellite for exporting content.
MCT0370 - Red Hat Satellite | 1 | Disconnected Satellite
RH00008 - Red Hat Enterprise Linux with Smart Management (Physical or VirtualNodes) | 50 | RHEL + Smart Management Subscriptions for the managed Nodes.


<INSERT IMAGE>

Firstly, we'd head over to the [subscription allocations](https://access.redhat.com/management/subscription_allocations) page to generate a subscription manifest.

We'll add the **Red Hat Satellite** subscriptions and the **Red Hat Enterprise Linux** subscriptions to the allocation, by selecting the **subscriptions** tab -> **add subscriptions**. If you have other subscriptions that you wish to use in the disconnected environment, add those here as well.


Note: we are **NOT** adding the **Red Hat Satellite Starter Pack** subscription.  This will be used by the connected Satellite to register to Red Hat Subscription Management (RHSM) for errata and software.  

![alt text](./images/add_sub_to_manifest.png "Adding Subscription to Manifest")


Lastly, download the subscription manifest:

![alt text](./images/download_manifest.png "Downloading Manifest")



### Building the connected Satellite

It is expected that the connected Satellite is built as per the specifications in the Installation Guide, taking into account the _Changes to the standard installation_ section above.

For the purposes of this example, our connected Satellite will be named `connected.example.com`

Firstly, let's register the Satellite to the Customer Portal

~~~
[root@connnected ~]# subscription-manager register
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: [REDACTED]
Password:
The system has been registered with ID: b9da2597-34db-40ca-a400-2c1e9741d846
~~~

Next, find the Satellite Starter pack subscription

~~~
[root@connnected ~]# subscription-manager list --all --available --matches '*Red Hat Satellite Starter Pack*'
+-------------------------------------------+
    Available Subscriptions
+-------------------------------------------+
Subscription Name:   Red Hat Satellite Starter Pack (manage up to 50 RHEL Instances)
Provides:            Red Hat Satellite Capsule Beta
                     Red Hat Software Collections (for RHEL Server)
                     Red Hat Satellite Capsule
                     Red Hat Satellite with Embedded Oracle
                     Red Hat Beta
                     Red Hat Satellite Beta
                     Red Hat Satellite 6 Beta
                     Red Hat Satellite 5 Managed DB Beta
                     Red Hat Enterprise Linux Server
                     Red Hat Enterprise Linux High Availability (for RHEL Server)
                     Red Hat Satellite
                     Red Hat Software Collections Beta (for RHEL Server)
                     Red Hat Satellite 5 Managed DB
                     Red Hat Enterprise Linux Load Balancer (for RHEL Server)
SKU:                 MCT1650
Contract:            11528077
Pool ID:             8a99f98261217cc5016122bc17bd0d5d
Provides Management: Yes
Available:           1
Suggested:           1
Service Level:       Premium
Service Type:        L1-L3
Subscription Type:   Standard
Ends:                01/22/2019
System Type:         Physical
~~~

And attach it:

~~~
[root@connnected ~]# subscription-manager attach --pool=8a99f98261217cc5016122bc17bd0d5d
Successfully attached a subscription for: Red Hat Satellite Starter Pack (manage up to 50 RHEL Instances)
~~~

Next, let's setup our repositories

~~~

subscription-manager repos --disable "*"
subscription-manager repos  \
  --enable=rhel-7-server-rpms \
  --enable=rhel-server-rhscl-7-rpms \
  --enable=rhel-server-7-satellite-6-beta-rpms \
  --enable rhel-7-server-satellite-maintenance-6-rpms
subscription-manager release --unset


~~~

And install the Satellite software and `foreman-maintain`, the Satellite maintenance tool.

~~~
yum clean all
yum makecache
yum install satellite -y
yum install rubygem-foreman_maintain -y
yum update -y
~~~

Next we'll install Satellite, setting our default organization to **RedHat**, default location to **RDU** and default admin password to **redhat** (adjust to your needs)
~~~
satellite-installer --scenario satellite -v \
  --foreman-initial-organization RedHat \
  --foreman-initial-location RDU \
  --foreman-admin-password redhat
~~~

Next, we'll setup our `.bashrc` and hammer config, so that we don't have to provide these parameters on every invocation of the CLI.

~~~
echo "ORG=RedHat" >> ~/.bashrc
echo "LOCATION=RDU" >> ~/.bashrc
echo "DOMAIN=example.com" >> ~/.bashrc
echo "SATELLITE=$(hostname -f)" >> ~/.bashrc
source ~/.bashrc

mkdir ~/.hammer/
cat > ~/.hammer/cli_config.yml<<EOF
:foreman:
    :host: 'https://$(hostname)/'
    :username: 'admin'
    :password: 'redhat'

EOF

~~~


Next, copy the subscription manifest to the Satellite. Then import it:
~~~
hammer subscription upload --organization "$ORG" --file /root/manifest.zip
~~~

Next,we'll update some parameters to make our disconnected workflows a bit easier. Firstly, we need to set our default download policy to `immediate`. This ensures that all of the content needed for our content exports is synchronized prior to our exports. If the policy is set to `on_demand` (Satellite 6.3's default) or `background`, the content export will be delayed until all of the remaining RPMs are downloaded from the CDN. Let's just set this now and get it out of the way. More on download policies can be found in [Satellite 6.2 Feature Overview: Lazy Sync](https://access.redhat.com/articles/2695861)

~~~
hammer settings set --name default_download_policy --value immediate
~~~

Also, for our Satellite `connected.example.com`, we have a dedicated partition `/exports`, which we are using for content exports. We'll need to update Satellite to use this directory:

~~~
hammer settings set --name pulp_export_destination --value /exports/
~~~

And make sure the SELinux contexts and permissions are correct.
~~~
chcon --verbose --recursive --reference /var/lib/pulp/katello-export/ /exports/
chmod --verbose --recursive --reference /var/lib/pulp/katello-export/ /exports/
chown --verbose --recursive --reference /var/lib/pulp/katello-export/ /exports/
~~~


### Enabling and Synchronizing Content

Now that we have our internet connected Satellite setup properly, let's enable and synchronize some repositories. We need to download at a minimum:

- Red Hat Enterprise Linux  (+ the Optional and Extras repos)
- Red Hat Enterprise Linux Kickstart repositories (to provision systems)
- Red Hat Satellite Tools (client tools)
- Red Hat Satellite Maintenance repository
- Red Hat Satellite
- Red Hat Software Collections (dependency for Satellite)

Enable any **other** repositories that will be needed in the disconnected environment here.

~~~
hammer repository-set enable --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server' \
  --basearch='x86_64' \
  --releasever='7Server' \
  --name 'Red Hat Enterprise Linux 7 Server (RPMs)'  

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server' \
  --basearch='x86_64' \
  --releasever='7Server' \
  --name 'Red Hat Enterprise Linux 7 Server - Optional (RPMs)'  

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server' \
  --basearch='x86_64' \
  --name 'Red Hat Enterprise Linux 7 Server - Extras (RPMs)'  

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server' \
  --basearch='x86_64' \
  --releasever='7.4' \
  --name 'Red Hat Enterprise Linux 7 Server (Kickstart)'  

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server' \
  --basearch='x86_64' \
  --name 'Red Hat Satellite Tools 6 Beta (for RHEL 7 Server) (RPMs)'  

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server' \
  --basearch='x86_64' \
  --name 'Red Hat Satellite Maintenance 6 (for RHEL 7 Server) (RPMs)'  

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Satellite' \
  --basearch='x86_64' \
  --name 'Red Hat Satellite 6.3 (for RHEL 7 Server) (RPMs)'

hammer repository-set enable \
  --organization "$ORG" \
  --product 'Red Hat Software Collections for RHEL Server' \
  --basearch='x86_64' \
  --releasever=7Server \
  --name 'Red Hat Software Collections RPMs for Red Hat Enterprise Linux 7 Server'

~~~

Now, let's synchronize them.
~~~
hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server'  \
  --name  'Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server'\
 --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server'  \
  --name  'Red Hat Enterprise Linux 7 Server - Optional RPMs x86_64 7Server' \
  --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server'  \
  --name  'Red Hat Enterprise Linux 7 Server - Extras RPMs x86_64' \
  --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server'  \
  --name  'Red Hat Enterprise Linux 7 Server Kickstart x86_64 7.4' \
  --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server'  \
  --name  'Red Hat Satellite Tools 6.2 for RHEL 7 Server RPMs x86_64' \
  --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Enterprise Linux Server'  \
  --name  'Red Hat Satellite Maintenance for RHEL 7 Server RPMs x86_64' \
  --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Satellite'  \
  --name  'Red Hat Satellite 6.3 for RHEL 7 Server RPMs x86_64' \
  --async

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Red Hat Software Collections for RHEL Server' \
  --name 'Red Hat Software Collections RPMs for Red Hat Enterprise Linux 7 Server x86_64 7Server' \
  --async
~~~

Next, we need to enable and synchronize a 3rd party repository. In this case we are using [EPEL](https://fedoraproject.org/wiki/EPEL)

~~~
wget -q https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7 \
  -O /root/RPM-GPG-KEY-EPEL-7

hammer gpg create \
  --key /root/RPM-GPG-KEY-EPEL-7  \
  --name 'GPG-EPEL-7' \
  --organization "$ORG"    

hammer product create \
  --name='Extra Packages for Enterprise Linux' \
  --organization "$ORG" \
  --description 'Extra Packages for Enterprise Linux'

hammer repository create \
  --name='EPEL 7 - x86_64' \
  --organization "$ORG"
  --product='Extra Packages for Enterprise Linux' \
  --content-type='yum' \
  --publish-via-http=true \
  --url=http://dl.fedoraproject.org/pub/epel/7/x86_64/  \
  --checksum-type=sha256 \
  --gpg-key=GPG-EPEL-7

hammer repository synchronize \
  --organization "$ORG" \
  --product 'Extra Packages for Enterprise Linux'  \
  --name  'EPEL 7 - x86_64'
~~~

At this point we have Red Hat Enterprise Linux & EPEL synchronizing, we can prepare to export it, but first a word from our CDN overlords.


### Architecture of the Red Hat CDN (and why it matters to a disconnected user of Satellite)

The Red Hat Content Delivery Network (henceforth known as the CDN) is the source of Red Hat content for Satellite 6. Understanding the structure of the Red Hat CDN is absolutely **critical** to properly supporting a disconnected Satellite.

It is important enough that I'll say it again,

~~~
Understanding the structure of the Red Hat CDN is absolutely
**critical** to properly supporting a disconnected Satellite.
~~~

#### What is the Red Hat CDN?

The Red Hat Content Delivery Network, nominally accessed via cdn.redhat.com is a geographically distributed series of static webservers, which contain content and errata that is designed to be consumed by systems. This content can be consumed directly (such as via a system registered via Red Hat Subscription Management) OR mirrored via on premise solution, such as Red Hat Satellite 6. The Red Hat Content Delivery network is protected by x.509 certificate authentication, to
ensure that only valid users can access it.

In the case of a system registered to Red Hat Subscription Management, the attached subscriptions govern which subset of the CDN the system can access. In the case of Satellite 6, the subscriptions that are attached to the subscription manifest govern which subset of the CDN the system can access.

#### Directory Structure of the CDN.

Now that you understand what the CDN is, let's take a look at a CDN mirror to see what the directory structure looks like:

~~~
$ tree -d -L 11
└── content
    ├── beta
    │   └── rhel
    │       └── server
    │           └── 7
    │               └── x86_64
    │                   └── sat-tools
    │                       └── 6
    └── dist
        └── rhel
            └── server
                └── 7
                ├── 7.4
                │   └── x86_64
                │       └── kickstart
                └── 7Server
                    └── x86_64
                        └── os

~~~

This directory structure is important and has the following meaning

- Top-level directory (always named _content_)
  - Second Level Directory (What is the lifecycle of this content? Common directories include _beta_ (for Beta code), _dist_ (for Production Bits) and _eus_ (For Extended Update Support bits))
    - Third Level Directory (which product. Usually _rhel_ for Red Hat Enterprise Linux)
      - Fourth Level Directory (Which Variant of the product. For Red Hat Enterprise Linux this includes _server_, _workstation_, and _computenode_ )
        - Fifth Level Directory (Major version, such as _5_,_6_, or _7_)
          - Sixth Level Directory (Release version  such as _7.0_, _7.1_, and _7Server_)
            - Seventh Level Directory (Base architecture, such as _i386_ or _x86_64_ )
              - Eighth Level Directory (repository name such as _kickstart_, _optional_, _rhscl_, etc). Some components have additional subdirectories, and those may vary.

This directory structure is also used in the subscription manifest. We can look at a subscription manifest to determine which directories of the CDN each subscription has access to.

**NOTE:** The output below has been shortened for brevity.

~~~
rct cat-manifest export.zip
+-------------------------------------------+
	Manifest
+-------------------------------------------+

General:
	Server:
	Server Version: 0.9.51.15-1
	Date Created: 2016-09-14T14:27:26.081+0000
	Creator: [REDACTED]

Consumer:
	Name: [REDACTED]
	UUID: [REDACTED]
	Type: satellite
Subscription:
	Name: Red Hat Enterprise Linux Server, Standard (Physical or Virtual Nodes)
	Quantity: 100
	Created: 2016-07-06T00:42:43.000+0000
	Start Date: 2016-07-04T04:00:00.000+0000
	End Date: 2017-07-04T03:59:59.000+0000
	Service Level: Standard
	Service Type: L1-L3
	Architectures: x86_64,ppc64le,ppc64,ia64,ppc,s390,x86,s390x
	SKU: RH00004
	Contract: [REDACTED]
	Order: [REDACTED]
	Account: [REDACTED]
	Virt Limit:
	Requires Virt-who: False
	Entitlement File: export/entitlements/8a99f98355af32300155bda82c721c90.json
	Certificate File: export/entitlement_certificates/4019380680377493250.pem
	Certificate Version: 3.2
	Provided Products:
		69: Red Hat Enterprise Linux Server
		176: Red Hat Developer Toolset (for RHEL Server)
		180: Red Hat Beta
		201: Red Hat Software Collections (for RHEL Server)
		205: Red Hat Software Collections Beta (for RHEL Server)
		240: Oracle Java (for RHEL Server)
		271: Red Hat Enterprise Linux Atomic Host
		272: Red Hat Enterprise Linux Atomic Host Beta
		273: Red Hat Container Images
		274: Red Hat Container Images Beta
		317: dotNET on RHEL (for RHEL Server)
		318: dotNET on RHEL Beta (for RHEL Server)
	Content Sets:
		/content/dist/rhel/server/5/$releasever/$basearch/cf-tools/1/os
		/content/dist/rhel/server/5/$releasever/$basearch/cf-tools/1/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/debug
		/content/dist/rhel/server/5/$releasever/$basearch/devtoolset/2/debug
		/content/dist/rhel/server/5/$releasever/$basearch/devtoolset/2/os
		/content/dist/rhel/server/5/$releasever/$basearch/devtoolset/2/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/devtoolset/debug
		/content/dist/rhel/server/5/$releasever/$basearch/devtoolset/os
		/content/dist/rhel/server/5/$releasever/$basearch/devtoolset/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/iso
		/content/dist/rhel/server/5/$releasever/$basearch/kickstart
		/content/dist/rhel/server/5/$releasever/$basearch/oracle-java/iso
		/content/dist/rhel/server/5/$releasever/$basearch/oracle-java/os
		/content/dist/rhel/server/5/$releasever/$basearch/oracle-java/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/os
		/content/dist/rhel/server/5/$releasever/$basearch/productivity/debug
		/content/dist/rhel/server/5/$releasever/$basearch/productivity/os
		/content/dist/rhel/server/5/$releasever/$basearch/productivity/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/rh-common/debug
		/content/dist/rhel/server/5/$releasever/$basearch/rh-common/iso
		/content/dist/rhel/server/5/$releasever/$basearch/rh-common/os
		/content/dist/rhel/server/5/$releasever/$basearch/rh-common/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/rhev-agent/3/debug
		/content/dist/rhel/server/5/$releasever/$basearch/rhev-agent/3/os
		/content/dist/rhel/server/5/$releasever/$basearch/rhev-agent/3/source/SRPMS
		/content/dist/rhel/server/5/$releasever/$basearch/rhn-tools/debug
		/content/dist/rhel/server/5/$releasever/$basearch/rhn-tools/iso
		/content/dist/rhel/server/5/$releasever/$basearch/rhn-tools/os
		/content/dist/rhel/server/5/$releasever/$basearch/rhn-tools/source/SRPMS
		/content/dist/rhel/server/6/$releasever/$basearch/cf-tools/1/debug
		/content/dist/rhel/server/6/$releasever/$basearch/cf-tools/1/os
		/content/dist/rhel/server/6/$releasever/$basearch/cf-tools/1/source/SRPMS
		/content/dist/rhel/server/6/$releasever/$basearch/debug
		/content/dist/rhel/server/6/$releasever/$basearch/devtoolset/2/debug
		/content/dist/rhel/server/6/$releasever/$basearch/devtoolset/2/os
		/content/dist/rhel/server/6/$releasever/$basearch/devtoolset/2/source/SRPMS
		/content/dist/rhel/server/6/$releasever/$basearch/devtoolset/debug
		/content/dist/rhel/server/6/$releasever/$basearch/devtoolset/os
		/content/dist/rhel/server/6/$releasever/$basearch/devtoolset/source/SRPMS
		/content/dist/rhel/server/6/$releasever/$basearch/insights-client/1/debug
		/content/dist/rhel/server/6/$releasever/$basearch/insights-client/1/os
		/content/dist/rhel/server/6/$releasever/$basearch/insights-client/1/source/SRPMS
		/content/dist/rhel/server/7/$releasever/$basearch/rhn-tools/debug
		/content/dist/rhel/server/7/$releasever/$basearch/rhn-tools/iso
		/content/dist/rhel/server/7/$releasever/$basearch/rhn-tools/os
		/content/dist/rhel/server/7/$releasever/$basearch/rhn-tools/source/SRPMS
		/content/dist/rhel/server/7/$releasever/$basearch/rhs-client/debug
		/content/dist/rhel/server/7/$releasever/$basearch/rhs-client/os
		/content/dist/rhel/server/7/$releasever/$basearch/rhs-client/source/SRPMS
		/content/dist/rhel/server/7/$releasever/$basearch/rhscl/1/containers
		/content/dist/rhel/server/7/$releasever/$basearch/rhscl/1/debug
		/content/dist/rhel/server/7/$releasever/$basearch/rhscl/1/iso
		/content/dist/rhel/server/7/$releasever/$basearch/rhscl/1/os

~~~

#### Listing files

Every directory from the Top-Level Directory (_content_) to the Eighth Level Directory (_optional_, _rhscl_ , etc) **MUST** contain a plain-text file, named **listing**, which contains, one per line, a listing of the subdirectories that the current directory contains. These are used by Red Hat Satellite 6 to determine which content does the CDN contain. See Below:

~~~
[root@cdn content]# ls
beta  dist  eus  listing
[root@cdn content]# cat listing
beta
dist
eus

[root@cdn content]# cd dist
[root@cdn dist]# ls
cf-me  listing  rhel
[root@cdn dist]# cat listing
cf-me
rhel
~~~

If the listing files are incorrect, you will NOT see the proper repositories as being available for synchronization.

#### Connected versus Disconnected Satellites.

The process of synchronizing a Satellite is exactly the same for disconnected versus connected Satellite users. The *only* noticeable difference between a connected Satellite (which can reach cdn.redhat.com) and a disconnected Satellite (which cannot) is:

- The disconnected satellite has a CDN URL defined which is NOT cdn.redhat.com
- The administrator has to create a local mirror of cdn.redhat.com that the Satellite can synchronize from.


### Understanding the Inter Satellite Sync capabilities within Satellite

Inter-Satellite Sync (link to 6.2 ISS Feature Overview), introduced the ability to export RPM repository content in one of two ways:

- Export a single repository
- Export a specific version of a content view (including the **Default Organization View**)

Depending on your use case, you will use either or both of these capabilities. When exporting non-Red Hat content, it is generally advisable to export on a per-repository basis.

When exporting Red Hat content it is expected that one exports a content view. There are a couple of reasons for this:

- As a content view is a collection of repositories, exporting a content view allows you to export all of the repositories as a unit.
- Content views are exported with all of the correct _listing_ files and as such is directly usable as a mirror of cdn.redhat.com.

**Which content view to use?**

You can use _any_ content view as a source of content for a disconnected Satellite. Under most circumstances you want to use the **Default Organization View**, which is the representation of the repositories which are currently downloaded within the organization.

This is the preferred way to export Red Hat content for usage with another Satellite. While you _can_ export a content view of your own creation, it is to be noted that :

- what is being exported is the content after filtering and publication.
- the content view's definition (and filters) are not exported.
- The relationship between an exported content view and a Satellite organizations CDN URL is 1:1

If you'd wanted to create (for example) two content views, one representing a filtered build of RHEL6, and another representing a filtered build of RHEL7, you'd have to export them individually and then combine them into a singular export. This advanced usage is beyond the scope of this document.

Now that you understand how the architecture of the Red Hat CDN, let's go export some content. 
