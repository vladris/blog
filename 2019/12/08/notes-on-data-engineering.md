# Notes on Data Engineering

For over a year now I've been the architect for the Customer Growth and
Analytics team in Azure, scaling out our big data platform as our team
grows and matures. I'm going to share a few observations on some of the
main problems we've been solving, problems which I believe are fairly
universal: I attended the [Strata Data
Conference](https://conferences.oreilly.com/strata-data-ai) this year
and speakers from many other companies were talking about similar
problems and the solutions they were implementing.

Not too long ago, the major challenges were around storing and
processing data at scale. Since this has been more or less commoditized
in the past few years, especially with the emergence of cloud providers,
it's interesting to think about how the big data landscape evolved and
what are some of the present challenges. I list some of them below.

## Compliance

Proper handling of data assets is a top priority for Microsoft and
should be for everyone. There are several aspects I consider to be part
of compliance.

First, there are regulatory obligations, probably the best-known example
being GDPR. In order to be compliant with GDPR, a data platform needs to
have the ability to "forget" about a user if the user so desires. A
data platform needs to support all capabilities required by regulations
of the countries it gets its data from. There are many other
regulations, and new ones can come up at any time. Staying compliant is
maybe the most important work for a data platform.

Next, there is access control: who gets to see what data. There are
various types of data for which access should be restricted. Personally
Identifiable Information (PII) is data that can be used to identify a
particular person, like name or email address. High Business Impact
(HBI) is data relevant to the business, like revenue numbers, devices
sold etc. Different companies use different taxonomies to classifies
their data assets, but regardless, in the big data space, it is a non
trivial problem to ensure that only people who are allowed to access a
certain data set are able to do so. If access to sensitive data sets is
under a need-to-know basis, and we create one security group per
dataset, just managing those security groups is hard in itself. On top
of that, people move and organizations change and that can impact who
has access to the data too.

I believe there is a lot of room to improve and innovate in this space.
There are many existing solutions to handle both regulatory and access
control requirements, but nothing that feels like *it just works*, and
definitely little in terms of industry standards.

## Metadata

As the data volume grows, organizing it becomes a big challenge. There
is a need for an additional layer of data about the data, or metadata.
There are several very important pieces of information which are not
readily available in the data itself that a big data platform must
provide.

First, there is simply the descriptions of various datasets: what the
various columns are, how often is the data refreshed, how fresh the data
is etc. There also needs to be an ability to search this metadata in
order to find relevant data available in the system.

Next, there is data lineage. Where did the data come from and how was it
sourced? This has compliance implications: for example if end users
agree to share telemetry data for the purpose of improving the product,
that data should not be used for other purposes.

Also for compliance purposes, various datasets and columns have to be
tagged as containing PII or other sensitive information so the system
can automatically lock down or purge this type of sensitive data when
needed.

Organizing data at scale also requires some amount of information
architecture. One aspect of this is controlled taxonomies: clear
definitions of what various business terms and data pieces mean, so
everyone working in the space shares the same understanding.

[Azure Data
Catalog](https://azure.microsoft.com/en-us/services/data-catalog/) is
the Azure offering in this space.

## Heterogeneity by Design

There is no one-size-fits-all data fabric. Each storage solution has
some workflows it was optimized for and some it's not so great at.
It's next to impossible to say that absolutely everything will be
running on one single solution, be that SQL, NoSQL, HDFS, or something
else. Some workflows need massive scale (processing terabytes of data),
some workflows need fast reads (serving a website). Teams upstream will
expose data from different storage solutions while teams downstream will
expect it in different storage solutions\...

Standardizing on a unique storage solution is unfeasible, so the next
best thing to do is to standardize on the tooling to move the data
around and ensure that it is easy to operate: make it easy to create a
new data movement pipeline, provide monitoring, alerting etc. Since data
movement is a necessity, it must be as reliable as possible.

Our team uses [Azure Data
Factory](https://azure.microsoft.com/en-us/services/data-factory/) for
orchestrating data movement at scale.

## DevOps

Another major bucket of work is bringing engineering rigor to workflows
supported by other disciplines like data science and machine learning
engineering. Again, with a small team, it is relatively easy to create
ad-hoc reports and run ad-hoc ML but this approach doesn't scale. Once
production systems depend on the output.

This is a solved problem in the software engineering field: source
control, code reviews, continuous integration and so on. But
non-engineering disciplines are not accustomed to this type of workflow
so there is definitely a need to educate, support, and create similar
devops workflows. Analytics and ML ultimately reduce to code (SQL
queries, Python, R etc.) and should be handled just as production code.

Our team supports these types of workflows using [Azure
DevOps](https://azure.microsoft.com/en-us/services/devops/) with
pipelines that can deploy ML and analytics from git to our production
environment.

## Data Quality

The last topic I will cover is data quality. The quality of all
analytics and machine learning outputs depends on the quality of the
underlying data.

There are multiple aspects to data quality. One set of definitions is
given by the article [Data Done Right: 6 Dimensions of Data
Quality](https://smartbridge.com/data-done-right-6-dimensions-of-data-quality/):

* Completeness - the dataset is not missing any required data.
* Consistency - the data is consistent across multiple datasets.
* Conformity - all data in the right format, within the right value
  ranges etc.
* Accuracy - the data accurately represents the domain being modelled.
* Integrity - the data is valid across all relationships and datasets.
* Timeliness - the data available when expected and datasets are not
  delayed.

A reliable data platform can run various types of data quality tests on
the managed datasets, both at scheduled times and during ingress. Issues
have to be reported and the overall state of the data quality made
visible through a dashboard so stakeholders can easily see which
datasets currently have problems and what are the potential implications
of that.

As of today, this seems to be a big gap in terms of industry-wide
standard solutions. Data engineering teams develop their bespoke data
test runners for their scenarios. There are many open source solutions,
but we don't have the equivalent of JUnit yet, nor a common language
for specifying tests and assertions.

## Conclusions

In the following years, I expect we will have both better tooling for
some of these problems and better defined industry-wide standards. As I
mentioned at the beginning of this post, not long ago, just storing and
processing huge amounts of data was a hard problem. Today, the main
challenges are around organizing and managing data. I fully expect that
in the near future we will have out-of-the-box solutions for all these
problems and a new set of challenges will emerge.
