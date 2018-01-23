---
title: User attributes
---

# User attributes

login.gov user accounts are either proofed (LOA3) or not (LAO1), corresponding to [NIST 800-63-2](http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-63-2.pdf) levels of assurance (LOA). Here are the possible attributes that can be requested at a given LOA.

| Attribute              | LOA1 | LOA3  | OpenID Connect | SAML |
| ---------------------- | ---- | ----- | -------------- | ---- |
| [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `sub` | `uuid` |
| Email                  | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `email` | `email` |
| First name             |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `given_name`             | `first_name`  |
| Middle name            |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> |                          | `middle_name` |
| Last name              |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `family_name`            | `last_name`   |
| Address line 1         |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `address.street_address` | `address1`    |
| Address line 2         |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `address.street_address` | `address2`    |
| City                   |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `address.locality`       | `city`        |
| State                  |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `address.region`         | `state`       |
| Zip code               |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `address.postal_code`    | `zipcode`     |
| Date of birth          |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `birthdate`              | `dob`         |
| Social security number |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `social_security_number` | `ssn`         |
| Phone                  |      | <img src="{{ site.baseurl }}/assets/img/check.svg" alt="checkmark"> | `phone`                  | `phone`       |
