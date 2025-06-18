
---
title: "Notifications Plugin"
id: index
weight: 1
description: How to get started with the notifications and signals
---
---

The Backstage Notifications System provides a way for plugins and external services to send notifications to Backstage users.

These notifications are displayed in the dedicated page of the Backstage frontend UI or by frontend plugins per specific scenarios.

Additionally, notifications can be sent to external channels (like email) via "processors" implemented within plugins.

Upstream documentation can be found in:

- [Notifications and Signals](https://backstage.io/docs/notifications/)
- [Backend email plugin](https://backstage.io/docs/reference/plugin-notifications-backend-module-email/)

# Frontend
Notifications are messages sent to either individual users or groups. They are not intended for inter-process communication of any kind. 

To list and manage, choose `Notifications` from the left-side menu item.

There are two basic types of notifications:
- Broadcast: Messages sent to all users of Backstage.
- Entity: Messages delivered to specific listed entities from the Catalog, such as Users or Groups.


![Frontend UI](./notificationsPage.png)

# Backend
The backend plugin provides the backend application for reading and writing notifications.

## Authentication
The Notifications are primarily meant to be sent by backend plugins. In such flow, the authentication is shared among them.

To let external systems (like a Workflow) create new notifications by sending POST requests to the Notification REST API, authentication needs to be properly configured via setting the `backend.auth.externalAccess` property of the `app-config` .

Refer to the service-to-service auth documentation for more details, focusing on the [Static Tokens](https://backstage.io/docs/auth/service-to-service-auth) section as the simplest setup option.

## Creating a notification by external services
An example request for creating a broadcast notification can look like:

```bash
curl -X POST https://[BACKSTAGE_BACKEND]/api/notifications -H "Content-Type: application/json" -H "Authorization: Bearer YOUR_BASE64_SHARED_KEY_TOKEN" -d '{"recipients":{"type":"broadcast"},"payload": {"title": "Title of broadcast message","link": "http://foo.com/bar","severity": "high","topic": "The topic"}}'
```

## Configuration
Configuration of the dynamic plugins is in the `dynamic-plugins-rhdh` ConfigMap created by the Helm chart during installation.

### Frontend configuration
Usually there is no need to change the defaults but little tweaks can be done on the props section:

```
            frontend:
              redhat.plugin-notifications:
                dynamicRoutes:
                  - importName: NotificationsPage
                    menuItem:
                      config:
                        props:
                          titleCounterEnabled: true
                          webNotificationsEnabled: false
                      importName: NotificationsSidebarItem
                    path: /notifications
```                    

### Backend configuration
Except setting authentication for external callers, there is no special plugin configuration needed.

# Forward to Email
It is possible to forward notification content to email address. In order to do that you must add the [Email Processor Module](https://github.com/backstage/backstage/tree/master/plugins/notifications-backend-module-email) to your Backstage backend. 

## Configuration
Configuration options can be found in [plugin's documentation](https://github.com/backstage/backstage/blob/master/plugins/notifications-backend-module-email/config.d.ts).

Example configuration:

```yaml
      pluginConfig:
        notifications:
          processors:
            email:
              filter:
                minSeverity: low
                maxSeverity: critical
                excludedTopics: []
              broadcastConfig:
                receiver: config # or none or users
                receiverEmails:
                  - foo@company.com
                  - bar@company.com
              cache:
                ttl:
                  days: 1
              concurrencyLimit: 10
              replyTo: email@company.com
              sender: email@company.com
              transportConfig:
                hostname: your.smtp.host.com
                password: a-password
                username: a-smtp-username
                port: 25
                secure: false
                transport: smtp
```

## Ignoring unwanted notifications
The configuration of the module explains how to configure filters. Filters are used to ignore notifications that should not be forwarded to email. The supported filters include minimum/maximum severity and list of excluded topics.

## User notifications
Each user notification has a list of recipients. The recipient is an entity in Backstage catalog. The notification will be sent to the email addresses of the recipients.

## Broadcast notifications
In broadcast notifications we do not have recipients, the notifications are delivered to all users.

The module's configuration supports a few options for broadcast notifications:
 - Ignoring broadcast notifications to be forwarded
 - Sending to predefined address list only
 - Sending to all users whose catalog entity has an email
