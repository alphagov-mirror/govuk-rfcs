# Links component in GOVUK Publishing components

## Summary

Create a HTML hyperlink component to standardise the use of links within GOVUK.

## Problem

Links styles across GOVUK are not consistent and often security factors are not considered when using links within GOVUK. This is often due to the lack of frontend developers available in teams. Sometimes security issues are not known by Developers and fixing them across multiple applications can add a lot of techincal debt - one recent example is the `target="_blank"` vulnerbility (https://www.manjuladube.dev/target-blank-security-vulnerability).

There are also certain styles that have been introduced in GOVUK that does not exist in the GOVUK Frontend Design system.

[Image of destructive link on Content publisher]

## Proposal

Add new hyperlink component into the GOVUK Publishing components.

The component should include:
- GOVUK Design system styles
- Security attributes for using different types of links
- Different types of link such as: Destructive, Inverse etc.
