+++
title = "Observability as code: dashboards"
date = 2020-09-10
+++

I recently received a really great question from a [Datadog](https://datadoghq.com) customer that I'd like to share concerning Observability as Code—specifically managing dashboards via [Terraform](https://www.terraform.io/) in this case, but the question and answer apply more broadly.


> While we use Terraform for 100% of our Datadog monitors, we've found [that] developers are less likely to update dashboards when they see a mistake in the UI, because they need to open a branch, run a pipeline and merge it in. This is a lot of overhead for a quick dashboard fix. Do you have any thoughts on this?
>
> SRE Team Lead



First, the standard disclaimer: There’s no one true way to do anything, and every recommendation or “best practice” out there needs to be weighed against the reality of the environment and organisation that you’re a part of. :)



That said, there are a few ways to look at this. If you’re part of a small shop with only a handful of resources, then it might be entirely reasonable to manage everything by hand—not just the dashboards. Scale, therefore, is a big part of the equation—and that notion extends to both computers and people. If you’re part of a small team that works either in isolation, or on things which are so specific as to be unusable to others, then here again, it’s entirely reasonable to manage those dashboards by hand. Still, even here one could make an argument for treating them as code: it’s self-documenting (for values of documentation), and it serves as a backup mechanism for when one of those manual edits inevitably goes awry.



As scale increases, so does the value of managing the dashboards as code. A prime example is repeatability. Many organisations end up deploying multiple iterations of very similar stacks. Creating the dashboards for these stacks by hand becomes tiresome very quickly. Furthermore, as the stack itself changes, having to go through each dashboard set to update them can be a nightmare (not to mention making sure that future stacks get the correct dashboard updates as well).



Some organisations may have more tightly controlled environments than others, and traceability for modifications is important. Even in environments that aren’t particularly controlled, there may be a healthy peer review culture that, far from being a burden, is a huge asset in ensuring that even small changes have more than one set of eyes in order to ensure completeness and suitability. In these cases, having tight feedback loops and rapid CI/CD mechanisms is paramount—if engineers feel like they’re fighting heavyweight tooling for “small changes”, then that’s a problem that should be addressed beyond the issue of dashboards specifically.



If you want to learn more about Datadog, Hashicorp Terraform, and Observability as Code, I invite you to check out our [upcoming joint webinar](https://www.datadoghq.com/partner/hashicorpwebinaremea/) on the topic on 23 September at 12:00 UTC. Hope to see you there!