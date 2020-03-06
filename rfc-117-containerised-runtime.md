# RFC 117: Use a container runtime

## Summary

Use a container runtime for hosting GOV.UK applications. This will provide a number of benefits, including:
* Removing our dependency on a version of Ubuntu (Trusty) which is in extended support (which ends in April 2022)
* Removing legacy infrastructure code which is difficult to maintain and extend
* Providing better value for money by using cloud resources more effectively
* Improving development velocity by reducing the difference between production and development environments
* Making it easier to adopt higher velocity deployment approaches (such as continuous deployment)

## Problem

GOV.UK currently hosts the majority of its applications and services using virtual machines running Ubuntu 14.04 LTS (Trusty Tahr).

Trusty is out of date, it was released in April 2014 and standard support ended in April 2019. At this time GOV.UK signed up for [Extended Security Maintenance](https://ubuntu.com/esm) (ESM) which provides continued security fixes for high and critical common vulnerabilities and exposures (CVEs) for the [most commonly used packages](https://wiki.ubuntu.com/SecurityTeam/ESM/14.04#A14.04_Infrastructure_ESM_Packages). ESM will end in April 2022.

Running an old operating system is preventing GOV.UK from upgrading key components of our technology stack (MongoDB, Puppet, Python, ...).

Upgrading Ubuntu in the existing infrastructure is difficult for a number of reasons, including but not limited to:

- Multiple applications share the operating system of the same virtual machine. This means it’s not easy to upgrade a single application at a time. Furthermore, the way our [Puppet code](http://github.com/alphagov/govuk-puppet/) is written makes supporting virtual machines on different operating systems at the same time hard to support.
- GOV.UK currently uses upstart as our init system, but newer versions of Ubuntu switch to systemd. This means GOV.UK will need to do a significant rewrite of our init scripts as part of an upgrade.
- Our current version of Puppet is 3.8, which is no longer supported (the current version is 6.13). Support for newer operating system versions is likely to be lacking in unsupported versions of Puppet.
- Other infrastructure as code tools in use (e.g. [Fabric](https://github.com/alphagov/fabric-scripts)) are also currently using very old versions which are very likely to have problems with newer operating systems.

## Proposal

GOV.UK will use a managed, container based hosting environment, such as [GOV.UK PaaS](https://www.cloud.service.gov.uk). The choice of hosting environment is out of scope for this RFC.

GOV.UK applications and their dependencies will be built into container images. This may be done using buildpacks or other means - this decision is also out of scope for this RFC.

A container runtime aligns with the [GOV.UK Infrastructure Architecture Goals for 2021](
https://docs.google.com/document/d/1ooN7wkYhEGvceGe9Qz_HNZa-GPtrjzK_vA4vfWYVn4c/edit#heading=h.cdrr7rv9t98f) to isolate applications through containers. This will give us better control of how resources are matched to applications.

The TechOps strategy is to use common components where available. GOV.UK has been following the strategy by using hosted versions of software ([ElasticSearch](https://aws.amazon.com/elasticsearch-service/) and [Postgres](https://aws.amazon.com/rds/)) and services ([Notify](https://www.notifications.service.gov.uk)) where available. Using a container runtime is a continuation of this policy.

GOV.UK has already containerised its development environment ([rfc-106](https://www.github.com/alphagov/govuk-rfcs/106)), a container runtime will narrow the gap between development and production.

### Benefits of this approach

The most immediate benefit of this approach is that it will allow us to incrementally upgrade the operating system version. By moving the applications into containers, GOV.UK can enable a much lower risk upgrade path.

Moving to a container based hosting environment will allow us to remove much of our legacy Puppet and Fabric code. This code is hard to maintain, and hard to hire people with the experience required to improve and upgrade. With industry and government overwhelmingly moving towards container based platforms, it should be much easier to hire people with these skills.

Containerised hosting environments can generally provide much better value for money in terms of resource usage, both because many small applications can be “bin-packed” onto the same infrastructure, and because it is much easier to autoscale applications.

Running containers in production allows for a reduction in the difference between development environments and production. This can lead to bugs being found more quickly, and faster development cycles.

Other improvements to GOV.UK’s development lifecycle (such as moving to continuous deployment) will be easier to implement on a container based infrastructure than they would in the current virtual machine based infrastructure.

### Risks of this approach

GOV.UK has not yet estimated the length of time this work will take, our current assumption is that it will take much less than 2 years to complete. There is a risk that the “unknown unknowns” GOV.UK discovers will prevent GOV.UK from finishing the containerisation work before support ends for Trusty. If this happens, GOV.UK may need to refocus our efforts to upgrade from Trusty in the existing infrastructure.

GOV.UK infrastructure has been in a transitory state for several years and containerisation will prolong this state. In addition in the short term GOV.UK will be adding complexity making supporting the platform hard for RE-GOV.UK and 2nd line.

The current migration (lift-and-shift to AWS) has been a complex and challenging piece of work. This has led to several engineers reporting feelings of burnout and exhaustion. If we fail to learn the lessons that GDS' past migration experiences have to offer there is a risk that doing this work could have significant negative consequences for the team, and for individuals' well-being. We should carefully consider how we structure teams, and how we can ensure that the teams have the time to do the work properly, get the training they need to learn new technologies, and have the agency they need to feel that they fully own their platform.

Focusing on containerisation will mean fewer people will be available to work on improving other parts of the platform.

GOV.UK has a significant amount of infrastructure outside of the applications themselves. In moving to a container based hosting platform, it’s possible that some existing behaviour may be missed, resulting in bugs.
### Risks of not doing this

GOV.UK reduces its options of how to mitigate Trusty reaching the end of support so the only option left is upgrading Ubuntu. This is potentially wasted work as once the upgrade is done GOV.UK will want to consider if running VMs is best for GOV.UK.

GOV.UK will also have to tackle a number of difficult tooling upgrades (Puppet, in particular) if GOV.UK wants to return to a situation where GOV.UK does not use unsupported dependencies.

GOV.UK will be harder for the rest of GDS to support, because its infrastructure will be significantly different. Almost all other service teams in GDS are using a container based hosting environment of some kind (mostly [CloudFoundry](https://www.cloudfoundry.org) or AWS ECS).
### Follow up work

GOV.UK will commit to doing these things:

- Investigate the practicalities involved in using different container runtimes to host GOV.UK (for example, how difficult will it be to run a hybrid containerised / non-containerised environment in [GOV.UK PaaS](https://www.cloud.service.gov.uk) vs. say [AWS ECS](https://aws.amazon.com/ecs/))
- Understand how containerising will affect our CI/CD, metrics & alerting, and support models.
