# 01 - Introduction

## Cloud Native Computing

Cloud native computing represents a modern approach to application development and deployment that maximizes the benefits of cloud infrastructure. When applications need to be accessible over the Internet, hosting them in the cloud becomes essential for scalability, reliability, and global reach.

A fundamental principle of cloud native architecture is the decoupling of applications from specific physical servers or hardware dependencies. This separation allows applications to run flexibly across different cloud resources without being tied to particular machines or infrastructure components.

When applications operate independently of specific servers, they require additional support systems to function effectively. Cloud native platforms must provide essential services that traditional server-based deployments would handle locally. These critical services include configuration management systems that allow applications to access necessary settings and parameters dynamically, persistent storage solutions that ensure data remains available even when application instances restart or move between servers, and application access mechanisms that enable secure communication between different components and external users.

Cloud native computing platforms are specifically designed to handle all these infrastructure concerns automatically. Rather than requiring developers to manually configure and manage these essential services, cloud native environments provide built-in solutions for configuration distribution, data persistence, and inter-application connectivity. This comprehensive approach allows developers to focus on application logic while the platform manages the underlying infrastructure complexities.

## Kubernetes and Cloud Native Computing

Kubernetes serves as the leading open-source platform that enables containerized applications to operate effectively within cloud native environments. This platform bridges the gap between traditional container deployment and the dynamic requirements of modern cloud infrastructure.

The core functionality of Kubernetes revolves around container orchestration, which involves intelligently managing where and how containers execute across distributed infrastructure. Rather than manually placing containers on specific machines, Kubernetes automatically determines optimal placement based on resource availability, application requirements, and system constraints.

One of Kubernetes' most valuable capabilities is its built-in scalability management. The platform continuously monitors application workloads and automatically adjusts the number of running container instances to match current demand. This dynamic scaling ensures applications can handle traffic spikes without manual intervention while optimizing resource utilization during quieter periods.

Kubernetes achieves true cloud native functionality through its comprehensive API that defines various resource types designed for distributed computing. Pods represent the smallest deployable units that can contain one or more containers, while Deployments manage the lifecycle and scaling of application instances. ConfigMaps provide a mechanism for storing and distributing configuration data throughout the cluster without hardcoding values into applications.

These resource abstractions work together to create an environment where applications operate independently of any specific physical servers. This server-agnostic approach allows workloads to move freely across the infrastructure, enabling the fault tolerance, scalability, and flexibility that define cloud native computing.

##  The 12-Factor Methodology: Building Resilient Cloud Applications

Traditional application deployment faces persistent obstacles that hinder reliability and scalability. Development teams struggle with environments that behave differently across stages, configuration files scattered throughout codebases, vertical scaling limitations that create bottlenecks, and deployment procedures that require extensive manual intervention and carry significant risk. The 12-Factor methodology emerged to address these pain points by establishing patterns that reduce environmental drift, facilitate team collaboration, and prevent the gradual degradation that affects long-lived applications. Originally developed by Heroku engineers based on their observations of successful cloud applications, these principles have become the cornerstone of modern application architecture. Understanding these factors is essential before diving into Kubernetes, as they guide how applications should be designed to fully leverage container orchestration platforms.

## 12-Factors Summary

Codebase has one repository used across all environments to keep software consistent. This prevents issues where code works differently in development and production.

Dependencies must be declared explicitly and isolated from the system. This ensures predictable builds and avoids hidden conflicts.

Configuration is stored in environment variables, separate from the code. This keeps credentials safe and allows the same code to run everywhere.

Backing services like databases or APIs are treated as replaceable resources. Switching from local to cloud services only requires updating configuration.

Build, release, and run are separate stages for deploying an app. This separation ensures consistent builds and enables quick rollbacks.

Processes must be stateless and store data externally. This design makes scaling and recovery simple.

Port binding allows apps to run with their own web server and expose services via a port. This makes deployment flexible and portable.

Concurrency means scaling by running multiple processes. Each type of process can be scaled independently.

Disposability requires apps to start fast and shut down cleanly. This supports smooth scaling and avoids data loss.

Development-production parity reduces differences between environments. It helps catch bugs early and simplifies deployments.

Logs are treated as event streams and sent to standard output. The environment manages storage and analysis instead of the app.

Admin processes like migrations run with the same code and settings as the app. This keeps them reliable and consistent.

## The Origins of Kubernetes

Kubernetes traces its roots back to Google Borg, an internal infrastructure platform that Google developed and utilized throughout the early 2000s. Borg served as Google's proprietary system for deploying and managing applications across their massive cloud-based infrastructure, handling the complex orchestration needs of services like Search, Gmail, and YouTube.

Drawing from over a decade of experience operating Borg at unprecedented scale, Google recognized the broader industry need for similar container orchestration capabilities. Rather than keeping this technology proprietary, Google made the strategic decision to create an open-source, platform-independent version that could benefit the entire technology community.

This initiative resulted in Kubernetes, which incorporated the lessons learned from Borg while being designed to work across different cloud providers and infrastructure platforms. The project represented Google's commitment to advancing cloud native computing beyond their own ecosystem.

Google officially unveiled Kubernetes in June 2014, marking the beginning of a new era in container orchestration. To ensure the project's long-term success and vendor neutrality, Google donated the Kubernetes source code to the Cloud Native Computing Foundation in 2015, establishing it as a truly community-driven open-source project that could evolve independently of any single company's interests.

## Understanding the Cloud Native Computing Foundation

The Cloud Native Computing Foundation operates as a subsidiary organization within the broader Linux Foundation ecosystem. Its primary mission centers on establishing and maintaining open-source standards that define cloud native computing practices across the industry.

The CNCF serves as a home for numerous projects that address various aspects of cloud native infrastructure. Among these projects, Kubernetes stands out as a flagship offering, providing the fundamental platform for orchestrating and running cloud native applications at scale.

## The Kubernetes Ecosystem and Distribution Options

The CNCF fosters an open collaborative environment where multiple projects can develop solutions for similar challenges within the cloud native space. This approach naturally leads to situations where different projects offer competing solutions for the same technical problems, giving users various options to choose from based on their specific needs.

When implementing Kubernetes in production environments, users face important decisions about which additional projects to incorporate alongside the core Kubernetes platform. These choices determine what specific functionality and capabilities their cloud native environment will provide.

Organizations can approach this selection process in two primary ways. The first approach involves manually selecting individual CNCF projects and integrating them with a basic Kubernetes installation, often referred to as vanilla Kubernetes. This method provides maximum flexibility but requires significant expertise to properly configure and maintain.

Alternatively, organizations can choose pre-packaged Kubernetes distributions that handle the complexity of project selection and integration automatically.

## Kubernetes Distribution Approaches

Kubernetes distributions represent complete, integrated solutions that combine the core Kubernetes platform with carefully selected CNCF projects to create fully functional cloud native environments. These distributions handle the complex task of ensuring compatibility and proper configuration between different components.

Distribution philosophies vary significantly in their approach to component selection. Opinionated distributions make specific technology choices for users by integrating only one CNCF project for each required functionality. This approach simplifies deployment and reduces decision-making complexity but limits user flexibility in component selection.

Open distributions take the opposite approach, providing users with choices about which specific solutions to implement for various functions. While this flexibility allows for greater customization, it also requires users to have more expertise in evaluating and selecting appropriate components.

Many distributions also include professional support services, making them suitable for mission-critical environments where reliability and assistance are essential requirements. This support can be crucial for organizations that need guaranteed response times and expert guidance for their Kubernetes deployments.

## Types of Kubernetes Distributions

The Kubernetes distribution landscape divides into several categories based on deployment models and target environments.

Cloud Managed Distributions integrate directly with public cloud platforms, providing seamless access to cloud-native services and infrastructure. Amazon's Elastic Kubernetes Service, Microsoft Azure's AKS, and Google's GKE represent the leading examples of this category. These services handle much of the operational complexity while providing deep integration with their respective cloud ecosystems.

On-premises Distributions focus on deployment within organizational data centers or private cloud environments. Google Anthos provides hybrid capabilities across multiple environments, while Rancher offers comprehensive cluster management features. Red Hat OpenShift brings enterprise-grade features and support, and Canonical Kubernetes provides a clean, upstream-focused approach to Kubernetes deployment.

Minimized Distributions serve development, testing, and learning purposes by providing lightweight Kubernetes environments that can run on individual developer machines or resource-constrained environments. Minikube offers the most straightforward local Kubernetes experience, K3s provides a production-ready lightweight option, and OpenShift Local delivers a complete Red Hat ecosystem experience for development work.

The scope and applicability of distributions extend beyond their initial design targets. Many distributions can adapt to different environments and use cases, allowing organizations to maintain consistency across development, testing, and production environments while adjusting configurations to meet specific requirements.