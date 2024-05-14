# Zero-downtime domain migration with the Firebase App Hosting REST API

The UX for "advanced config" for custom domains, which allows migration of existing sites without downtime, is coming as a fast-follow. The API, however, already works! This is a quick guide to migrate an apex domain with zero downtime.

To get started, [install the gcloud CLI](https://cloud.google.com/sdk/docs/install) and log into an account that has administrative permissions on your project. Set real values for the following environment variables:

```
PROJECT=your-project-here
LOCATION=us-central1
BACKEND=your-backend-name
DOMAIN=example.com

# This doesn't change
ORIGIN=https://firebaseapphosting.googleapis.com/v1beta
```

## 1. Register your domain

To create a domain in Firebase App Hosting, simply POST to the domains endpoint:

```
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  $ORIGIN/projects/$PROJECT/locations/$LOCATION/backends/$BACKEND/domains?domainId=$DOMAIN
```

This will return a long running operation link. Don't worry; you can get everything by just fetching the domain resource.

## 2. Fetch discovered state

Get the latest state of your domain in Firebase App Hosting by fetching the domain resource:

```
curl -X GET \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  $ORIGIN/projects/$PROJECT/locations/$LOCATION/backends/$BACKEND/domains/$DOMAIN
```

After our initial scan of your DNS records you'll see a list of instructions in `customDomainStatus.requriedDnsUpdates`. *DO NOT FOLLOW INSTRUCTIONS TO SET A OR CNAME RECORDS ON $DOMAIN YET* (that's what makes this a zero-downtime migration!)

## 3. Mint cert and pass ownership challenge

### Mint cert

The `requiredDnsUpdates` field has a set of records for a domain starting with `_acme-challenge` that will ask you to add a CNAME record. Adding that record allows App Hosting to generate SSL certificates for your domain. Once you add that record, your domain's `customDomainStatus.certState` field should be `CERT_ACTIVE` within about an hour, indicating that App Hosting has successfully generated a certificate.

#### Pass ownership challenge

App Hosting uses DNS records to determine which Backend's content to serve when it recieves requests for your domain. For zero-downtime migration, you'll need to add a TXT record instructing App Hosting to serve content for the custom domain you just created. The record data will be a string in the form `fah-claim=[token]`.

If you're adding an apex domain (e.g. foo.com) the `requiredDnsUpdates` field will have that TXT record under it. If you're migrating a subdomain (e.g. www.foo.com), the record won't be displayed in `requiredDnsUpdates`, but you can generate one yourself by replacing `[token]` with `002-02-` plus the contents of your domain's `uid` field. It should look something like this: `fah-claim=002-02-########-####-####-####-############`.

Once you add the TXT record, your domain's `customDomainStatus.ownershipState` should be `OWNERSHIP_ACTIVE` within about an hour, indicating that App Hosting will serve content for your custom domain's Backend when it recieves requests for your domain.

## 4. Wait

Occassionally re-check the status of your domain record. You're looking for `customDomainStatus.ownershipState: OWNERSHIP_ACTIVE` and `customDomainStatus.certState: CERT_ACTIVE`. Once you see this, Firebase App Hosting has verified your domain and provisioned your TLS certificate.

## 5. Set A record

*Now* it's safe to modify your A record(s) to start sending traffic to App Hosting. 

If you're regestring an apex domain (e.g. foo.com), you can simply add the A record displayed in the `customDomainStatus.requiredDnsUpdates.desired` record set. If you're adding a subdomain (e.g. www.foo.com), you'll have to find the correct A record for yourself; to do so, use a tool like [dig](https://toolbox.googleapps.com/apps/dig/) to query the CNAME record listed in `customDomainStatus.requiredDnsUpdates.desired`. It should return exactly 1 A record, which you should add to your domain. 

Be sure to also remove any A or AAAA records from your previous provider, to ensure all traffic is routed to App Hosting.

## 6. All done!

Once your A record change propagates, App Hosting will start receiving requests for your domain. Some time after that, the `customDomainStatus.hostState` will flip to `HOST_ACTIVE`. Don't worry if you start seeing App Hosting content on your domain before that state changes--`hostState` is purely informational, App Hosting's way of letting you know that it's seen your successful A record change.
