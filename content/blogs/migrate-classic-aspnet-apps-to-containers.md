---
author: Teddy
title: "Migrate Classic ASP.NET Apps to Containers"
date: 2023-12-28
#description: ""
summary: "The motivations, strategies, and challenges of migrating classic ASP.NET applications to containers"
tags: ["container", "mono", "aspnet", "dotnet", "devops"]
ShowToc: true
TocOpen: true
---
## Introduction

In the ever-evolving landscape of web development, staying current is not just an option but a necessity. Migrating classic ASP.NET applications to containers is one such journey that many organizations are undertaking to modernize their infrastructure. This blog post will explore the motivations, strategies, and challenges of migrating classic ASP.NET applications to containers.

## Motivations

1. **Dependency Management**: Classic ASP.NET applications often rely on specific server configurations and dependencies, making deployment and maintenance challenging. Containers encapsulate these dependencies, providing a consistent runtime environment across various stages of development.
2. **Scalability and Resource Efficiency**: Containers are lightweight and can be easily scaled up or down based on demand. This flexibility allows for optimal resource utilization and improved performance, especially in scenarios with varying workloads.
3. **Isolation and Security**: Containers provide isolation between applications and their dependencies. This isolation enhances security by reducing the attack surface and mitigating potential conflicts between different components.
4. **DevOps Integration**: Migrating to containers aligns with DevOps principles, allowing for streamlined development, testing, and deployment pipelines. Containers facilitate continuous integration and continuous deployment (CI/CD) practices, enabling faster and more reliable software delivery.
5. **Cost Savings on Windows OS Licenses**: If you can migrate classic ASP.NET applications to containers on Linux, it can significantly save costs by eliminating the need for expensive Windows Server licenses. Most AWS EC2 Linux resources cost 50% less comparing to the Windows ones.
6. **Scalability Across Cloud Providers**: Only limited cloud providers support Window containers, so if you can migrate your classic ASP.NET apps to Linux containers, you can also get the portability across different cloud providers, allowing your organization to choose the most cost-effective solution for their specific needs. This flexibility enhances scalability and resilience by enabling applications to run seamlessly across various cloud environments.

## Strategies

1. **Rehosting (Lift and Shift)**: This strategy involves packaging the existing ASP.NET application into a windows container without making significant code changes. While it provides a quick path to containerization, it may not fully leverage container-native features, though this approach provides the best compability. Also, like Windows VMs, Windows containers usually still cost double comparing to Linux containers because of the additional Windows OS licenses costs. And you have to also be aware that only limited cloud providers and DevOps tools support Window containers.
2. **Replatforming**: This strategy involves making minor modifications to the application to take advantage of container-specific features, such as dynamic scaling, running in Linux containers, etc. It strikes a balance between speed and optimization, allowing for incremental improvements. For most classic ASP.NET apps, with minor changes, they might be able to run in Linux containers with open source [MONO](https://www.mono-project.com/) runtime. But please be aware that the MONO runtime implementation is not widely used in web server world and its community and releasing is not very active, so it has some limitations and sometimes bugs for some components such as WCF hosting, in worst cases if you cannot work around some issues, you might have to fix the bugs by yourself. You need to carefully do performance testing, especially concurrent testing against your pages/services before deploying to production. If your applications are lucky enough of only using ASP.NET components, such as ASP.NET MVC, which are fully supported by ASP.NET Core runtime, it is highly recommended to migrate your apps to ASP.NET Core with small changes. ASP.NET Core is actively supported by latest .NET Core runtime, so its more performant and stable than the MONO runtime.
3. **Refactoring/rewriring**: In this approach, developers rewrite or re-architect parts or even the entire of the application to fully embrace containerization benefits. While time-consuming, this strategy can unlock the full potential of containers and modernize the application architecture. A more common strategy is, when #1 and #2 of the strategies can already help you migrate to containers and save costs, the next step, you can incrementally refactor/rewrite your apps to unlock the full benefits of containeration. This is also not limited to using .NET Core, based on your organization's knowledge and resources, you can consider any modern development languages and platforms such as nodejs, go, etc.

## Challenges

1. **Legacy Code Challenges**: Classic ASP.NET applications may have legacy code and dependencies that are not easily containerized. Rewriting or refactoring these components can be time-consuming and complex. The suggestions based on my experience is, if you cannot find a new version or direct replacement, consider using Saas services of container based solutions instead. For example, if you have a 3rd party lib doing PDF/Word file generation or GeoIP-to-country resolving, you can wrap a Saas service or a container based API as a lib instead.
2. **Learning Curve**: Containerization introduces a learning curve for development and operations teams unfamiliar with container technologies like Docker and Kubernetes. Training and skill development may be required.
3. **Lack of Native Support**: Classic ASP.NET was not designed with containerization in mind. Achieving seamless integration may require workarounds and additional configurations. So during your migration, it will be super helpful if you can have some experienced senior engineers or 3rd party company/freelencer as consultants.

## Some MONO Known Issues & Workarounds

1. The Mono System.Web assembly is [missing the class: System.Web.Security.AntiXss.AntiXssEncoder](https://github.com/mono/mono/issues/13147), workaround: use System.Web.HttpUtility.HtmlEncoder instead.
2. The async ASP.NET pipeline is not fully implemented in MONO, workaround, change all async actions to sync.
3. WsHttpBinding in MONO WCF not working, workaround: use BasicHttpBinding instead.
4. Some embbeded code syntaxes not work in aspx and asax pages, you will get runtime compilation errors, workaround, try to move the embbeded code to the code-behind file.
5. Windows and Linux have different timezone names, code such as TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time") will fail in linux at runtime, workaround: use the 3rd party TimeZoneConverter nuget package and the working code will be: TimeZoneConverter.TZConvert.GetTimeZoneInfo("Eastern Standard Time").
6. The MONO WCF System.ServiceModel implementation has some thread-safety and memory-leak issues, workaround: avoid using WCF if possible, or try to fix the code by yourself. There is [a fork of the mono project repo](https://github.com/ef-labs/mono) which contains some fixes made by my current company, but the fixes are as-is and only tested for components we use, you should always load test your migrated services when running in MONO.
7. The MONO WebForm pipeline implementation is not as strong as the MS .NET Framework impelementation under attacks or security scanning and there is no easy way to fix, workaround: enable WAF (Web Application Firewall) feature at your cloud provider to protect all your MONO backend services.
