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

After our initial scan of your DNS records you'll see a list of instructions in `customDomainStatus.requriedDnsUpdates`. *DO NOT FOLLOW INSTRUCTIONS TO SET A OR CNAME RECORDS YET* (that's what makes this a zero-downtime migration!)

## 3. Pass ownership challenge

Make the suggested TXT record change to your DNS config. This will prove to Firebase App Hosting that you own this domain and we should serve your traffic.

## 4. Wait

Occassionally re-check the status of your domain record. You're looking for `customDomainStatus.ownershipState: OWNERSHIP_ACTIVE` and `customDomainStatus.certState: CERT_ACTIVE`. Once you see this, Firebase App Hosting has verified your domain and provisioned your TLS certificate.

## 5. Set A record

*Now* it's safe to modify your A record according to `customDomainStatus.requiredDnsUpdates.desired`. Make the change and wait for DNS to propegate. Your site will migrate to Firebase App Hosting with zero downtime.