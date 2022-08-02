---
title: "Merchant Center SSO"
date: 2022-07-15T13:43:31+02:00
draft: false
---
We do use SSO with Juniqe Google Apps user database.
Auth0 is used as OIDC provider.

https://docs.commercetools.com/merchant-center/single-sign-on-beta

## Usage

* Open https://mc.commercetools.com/login/sso
* Enter organisation name "juniqe"
* In Auth0 opening dialog, click "Continue with JuniqeGoogleApps"
* In Sign in with Google, sign in into juniqe google account(gmail)
* You are in!

new users have no privilgies, Administrator supposed  to invite them to proper groups after first login

## Setup for new CT Org

1. Setup G Suite Enterprise connection in Auth0 (follow Auth0 instructions). If there are one, we don't need new one.
2. Setup Application in Auth0. We need 1 app per org.
3. Create CT team `new users` with no privilegies
4. Fill in Commerce tools SSO page with Auth0 config.

Example

* Authority URL: `https://ichbinjuniqe.eu.auth0.com/oauth/`
* Client ID: client id from auth0 app
* Team ID: `new users`
