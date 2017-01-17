# Large-scale cluster management at Google with Borg
> Google Inc.

Abstract

Google’s Borg system is a cluster manager that runs hundreds
of thousands of jobs, from many thousands of different
applications, across a number of clusters each with up to
tens of thousands of machines.
It achieves high utilization by combining admission control,
efficient task-packing, over-commitment, and machine
sharing with process-level performance isolation. It supports
high-availability applications with runtime features that minimize
fault-recovery time, and scheduling policies that reduce
the probability of correlated failures. Borg simplifies
life for its users by offering a declarative job specification
language, name service integration, real-time job monitoring,
and tools to analyze and simulate system behavior.
We present a summary of the Borg system architecture
and features, important design decisions, a quantitative analysis
of some of its policy decisions, and a qualitative examination
of lessons learned from a decade of operational
experience with it.

1. Introduction

The cluster management system we internally call Borg admits,
schedules, starts, restarts, and monitors the full range
of applications that Google runs. This paper explains how.
Borg provides three main benefits: it (1) hides the details
of resource management and failure handling so its users can
focus on application development instead; (2) operates with
very high reliability and availability, and supports applications
that do the same; and (3) lets us run workloads across
tens of thousands of machines effectively. Borg is not the
first system to address these issues, but it’s one of the few operating
at this scale, with this degree of resiliency and completeness.
This paper is organized around these topics, concluding with a set of qualitative observations we have made
from operating Borg in production for more than a decade.

2. The user perspective
Borg’s users are Google developers and system administrators
(site reliability engineers or SREs) that run Google’s
applications and services. Users submit their work to Borg
in the form of jobs, each of which consists of one or more
tasks that all run the same program (binary). Each job runs
in one Borg cell, a set of machines that are managed as a
unit. The remainder of this section describes the main features
exposed in the user view of Borg.


2.1 The workload
Borg cells run a heterogenous workload with two main parts.
The first is long-running services that should “never” go
down, and handle short-lived latency-sensitive requests (a
few µs to a few hundred ms). Such services are used for
end-user-facing products such as Gmail, Google Docs, and
web search, and for internal infrastructure services (e.g.,
BigTable). The second is batch jobs that take from a few
seconds to a few days to complete; these are much less sensitive
to short-term performance fluctuations. The workload
mix varies across cells, which run different mixes of applications
depending on their major tenants (e.g., some cells are
quite batch-intensive), and also varies over time: batch jobs
come and go, and many end-user-facing service jobs see a
diurnal usage pattern. Borg is required to handle all these
cases equally well.
A representative Borg workload can be found in a publiclyavailable
month-long trace from May 2011 [80], which has
been extensively analyzed (e.g., [68] and [1, 26, 27, 57]).
Many application frameworks have been built on top of
Borg over the last few years, including our internal MapReduce
system [23], FlumeJava [18], Millwheel [3], and Pregel
[59]. Most of these have a controller that submits a master
job and one or more worker jobs; the first two play a similar
role to YARN’s application manager [76]. Our distributed
storage systems such as GFS [34] and its successor CFS,
Bigtable [19], and Megastore [8] all run on Borg.
For this paper, we classify higher-priority Borg jobs as
“production” (prod) ones, and the rest as “non-production”
(non-prod). Most long-running server jobs are prod; most
batch jobs are non-prod. In a representative cell, prod jobs
are allocated about 70% of the total CPU resources and represent
about 60% of the total CPU usage; they are allocated
about 55% of the total memory and represent about 85% of
the total memory usage. The discrepancies between allocation
and usage will prove important in §5.5.
