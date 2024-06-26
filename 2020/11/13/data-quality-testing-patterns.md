# Data Quality Testing Patterns

The insights generated by a data platform are only as good as the
quality of the underlying data. I briefly mentioned data quality in my
[Notes on Data
Engineering](https://vladris.com/blog/2019/12/08/notes-on-data-engineering.html)
blog post and elaborated on the DataCop solution in the [Partnering for
data
quality](https://medium.com/data-science-at-microsoft/partnering-for-data-quality-dc9123557f8b)
Medium article I co-authored with my colleagues.

In this post I want to talk about some of the requirements and common
patterns of data quality test solutions. I maintain that in the future
data quality testing will be commoditized and offered "as a service"
by cloud providers. As of today, it is still something we have to stitch
together or onboard a 3rd party solution. Below is a blueprint for such
solutions.

## Data Fabrics

Any data quality test run is eventually translated into a query
executing on the underlying data fabric. In Azure, this can be, for
example, Azure SQL, Azure Data Explorer, Data Lake Storage etc. Data
quality frameworks need to support multiple data fabrics for several
reasons. First, in a big enough enterprise, data will live in multiple
systems. Second, different data fabrics are optimized for different
workloads, so even if we are very strict about the number of technology
choices, sometimes it simply makes sense to leverage different data
fabrics for different workloads.

We probably don't want to manage multiple data quality solutions
specializing in different data fabrics, since data quality solutions
become part of our infrastructure. Regardless of target data fabrics,
any data quality solution needs to implement some form of test
scheduling, alerting etc.

The best pattern is to support multiple data fabrics through a plug-in
model, so support for additional data fabrics can be easily added by
implementing a new plug-in, while the core of the system is shared for
all data fabrics plugged into the system.

## Types of Tests

The [6 Dimensions of Data
Quality](https://smartbridge.com/data-done-right-6-dimensions-of-data-quality/)
article talks about *completeness*, *consistency*, *conformity*,
*accuracy*, *integrity*, and *timeliness*. I think of data quality
testing like unit testing for code, so I will present a slightly
different take on this. Some of the dimensions are harder to test for
than others, for example checking consistency across multiple data
systems is non-trivial (more equivalent to an integration test). Some of
the dimensions have nuances -- for example detecting anomalies in day
over day data volume change to identify potential issues vs. just
looking at a point in time.

### Availability

The simplest type of data test is availability, meaning *is data
available for a certain date*. If, for example, we ingest yesterday's
telemetry data every night, we expect to have yesterday's telemetry
data available in our system tomorrow.

This type of test can be implemented as a query against the underlying
data fabric that ensures some data is there. That can mean a query that
returns at least one row passes the test. This can be an early smoke
test we can run before performing more involved testing.

### Correctness

This type of test ensures data is *correct*, based on some criteria. For
example, ensuring columns that shouldn't be empty are not empty, values
are within expected ranges, and so on.

This type of test can be implemented as a query against the underlying
data fabric that ensures no out-of-bounds values are returned. Going
back to the unit test equivalency, we likely want multiple correctness
tests per dataset, one or more per column we want to test.

### Completeness

Completeness tests expand on availability tests and validate not only
that some data is available, but that *all* the data is available.

In some cases, this can be an exact query: for example, if we are
expecting some data for all 50 states of US, we can check that the count
of distinct states is 50.

More advanced checks look at historical data and ensure the volume of
the data we are checking is within a certain threshold -- for example,
our telemetry data volume should be within +/-5% of the data volume we
observed yesterday.

### Anomaly detection

We won't go to deep into this, but there are some complexities
associated with the previous type of historical data checks. For
example, website traffic volume, depending on the website, might be very
different day over day between weekends and workdays, or during holidays
etc.

For these situations we can use more complex AI-based anomaly detection
to track volume over time until the system can identify anomalous data.

## Queries, Code or Configuration

The implementation of tests can be done as queries, code, configuration,
or a mix.

Implementing all tests as queries means writing each test as a stored
procedure (or equivalent), so it is fully implemented on the data fabric
it executes against. The test framework just invokes the test and reads
the result. The main challenge with this is that there is not a lot of
reuse. Since the framework ends up just calling a stored procedure, it
is up to the test author to write the test, which is ultimately an
arbitrary query.

Implementing all tests as code means wrapping each data quality test
into a test method in some programming language. The main advantage of
this is we can use an off-the-shelf test framework with all its
benefits. Unlike data quality testing, the field of software testing is
mature. On the other hand, the main drawback is that authoring tests as
code really raises a barrier to entry, making it harder for
non-engineers to create tests.

Tests as configuration means defining tests using text configuration
files in a format like JSON, YAML, or XML. The framework interprets
these and translates them into the final queries to execute against the
data fabric. The main advantage of this approach is a fairly data
fabric-independent way to specify tests, around which we can build
schema validation etc. The disadvantage is increased complexity in the
framework, as it must translate configuration to queries.

What works best, from my experience, is a mix of queries and
configuration: give enough flexibility in the config schema to author
custom queries and let the framework handle some of the common concerns
like scheduling.

## Test Execution

Another important concern is when we want to run a given data quality
test.

In some scenarios we want to run tests as part of a data movement
workflow. We want to either check that the data is in good shape at the
source, before we copy it, or check that the data is in good shape after
it got copied to the destination. Let's call this type of *execution on
demand*. This type of test execution is integrated in the ETL pipeline.

In other scenarios we want to run the tests on a schedule, as we expect
the data to be available. For example, if data should have arrived in
our SQL database by 7 AM every morning, we want to run our data quality
tests at 7 AM to make sure everything is in good shape. In this case, we
don't really care how the data gets here, so we are running independent
of the ETL pipeline that brings the data in. Let's call this *scheduled
execution*.

A good data quality test framework should support both on demand and
scheduled test execution, as there is room (and need) for both types of
tests.

## Monitoring and Alerting

A data quality framework must be integrated with an alerting system such
that whenever a data quality test fails, data engineers are proactively
alerted so they can start investigating/mitigating the data issue.

A dashboard showing the overall health of the data platform is another
important component. This should include data quality for all datasets
under test, so stakeholders suspecting a data issue can quickly check to
see when did the latest tests run for a given dataset and what were the
results.

A slightly more advanced capability is lineage tracking -- if we
identify an issue with a dataset, do we know what is the impact? What
analytics or machine learning models or downstream processing is
impacted? Tracking metadata like lineage is a deep topic itself but
integrating this with a data quality framework enables very powerful
observability.

## Summary

In this post we covered some patterns for data quality testing:

* Supporting multiple data fabrics via a plug-in model.
* Types of tests, from simple availability to anomaly detection.
* How to best specify tests as a mix of queries and configuration.
* Test execution, both on a schedule and on-demand.
* Monitoring and alerting to bubble up data quality issues.

A data quality test framework is a critical part of a data platform,
giving confidence in the results produced by all workloads running on
the platform.
