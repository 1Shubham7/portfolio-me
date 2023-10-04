---
title: "Open-Source 101: The basic must-know terms for an open-source beginner"
description: "APIs define a standardized way for different software components to interact, providing a layer of abstraction that simplifies development and integration. They typically consist of a collection of pre-defined functions, classes, or endpoints that developers can use to request or exchange data, perform operations, or access specific functionalities of the underlying system."
dateString: October 2023
draft: false
tags: ["API", "DevOps", "Open-source", "Backend"]
weight: 101
cover:
    image: "blog/api.png"
---

During our Web development journey will all get to hear the term - "API". Some of us must have used some APIs, but not many know about what API actually is and how to play around with API. Join me in this article where we dive deep into the world of API and understand the concepts of API from beginner to advanced.

## What is API?
The full form of API is Application Programming Interface. In simplest terms, API is a contract that allows code to talk to some other code. It serves as a bridge that enables a seamless exchange of data and functionality between various systems, enabling developers to access and utilize the features of another application or service without having to understand its internal workings.

> APIs define a standardized way for different software components to interact, providing a layer of abstraction that simplifies development and integration. They typically consist of a collection of pre-defined functions, classes, or endpoints that developers can use to request or exchange data, perform operations, or access specific functionalities of the underlying system.

## Why do we need API?
It's very basic, for making a weather app - would you launch satellites or just use a weather API?

Some more examples of why we need API are:

***Social Media Integration:*** Social media platforms like Facebook, Twitter, and Instagram provide APIs that allow developers to integrate their applications with these platforms. This integration enables users to log in using their social media accounts, share content, and access social features within the application.

***Payment Gateways:*** APIs offered by payment gateway providers such as PayPal, Stripe, and Braintree allow developers to incorporate secure payment processing functionality into their applications. This enables users to make online payments using different payment methods.

***Mapping and Geolocation Services:*** APIs provided by mapping and geolocation services like Google Maps and Mapbox enable developers to embed maps, geocoding, and routing functionality into their applications. These APIs provide features such as displaying locations, calculating distances, and finding directions.

***Weather Data:*** Weather APIs, such as those offered by OpenWeatherMap and Weather Underground, provide access to real-time and forecast weather data. Developers can utilize these APIs to display weather information within their applications, enabling users to check current weather conditions or plan for future events.

***E-commerce Platforms:*** APIs offered by e-commerce platforms like Shopify and WooCommerce allow developers to create online stores, manage inventory, process orders, and retrieve product information. These APIs enable seamless integration between e-commerce platforms and custom applications.

***Email Services:*** Email service providers like SendGrid and Mailchimp offer APIs that allow developers to send and manage emails programmatically. This enables applications to send transactional emails, newsletters, and marketing campaigns.

***Cloud Services:*** Cloud computing providers such as Amazon Web Services (AWS), Microsoft Azure, and Google Cloud Platform offer APIs for various services like storage, computing, and database management. Developers can leverage these APIs to build scalable and flexible cloud-based applications.

## API Architectures
There are several types of API architectures commonly used in web development. Here are three prominent ones:

### REST (Representational State Transfer): 
REST is a widely adopted architectural style for designing networked applications. RESTful APIs utilize the principles of the HTTP protocol and leverage its methods (GET, POST, PUT, DELETE, etc.) to perform operations on resources. REST APIs typically use URLs (Uniform Resource Locators) to identify resources and employ different HTTP status codes to indicate the outcome of a request. They emphasize statelessness, scalability, and interoperability, making them popular for building web services.

### SOAP (Simple Object Access Protocol):
SOAP is an XML-based protocol that enables communication between applications over a network. SOAP APIs define a strict structure for request and response messages using XML schemas. They rely on the XML format for data representation and typically use the POST method for communication. SOAP APIs often employ Web Services Description Language (WSDL) to describe the available operations, message formats, and service endpoints. SOAP APIs are known for their strong message-level security and support for advanced features such as transactions and reliability.

### GraphQL:
GraphQL is an open-source query language and runtime for APIs developed by Facebook. It allows clients to request precisely the data they need, eliminating over-fetching or under-fetching of data common in traditional REST APIs. With GraphQL, clients can send queries specifying the desired data structure, and the server responds with a JSON payload containing only the requested data. This flexible and efficient approach to data fetching makes GraphQL popular for applications with complex data requirements and enables clients to aggregate data from multiple sources in a single request.

These are just a few examples of API architectures, and there are other variations and hybrid approaches as well. The choice of API architecture depends on factors such as the project requirements, scalability needs, interoperability considerations, and the preferences of the development team.

## Conclusion
In conclusion, APIs are the lifeblood of modern web development, enabling seamless communication and integration between different applications and services. They provide a standardized way for software components to interact, allowing developers to leverage the functionality of external systems without needing to understand their internal complexities.

Throughout this article, we've explored the world of APIs from beginner to advanced levels. We've learned that APIs serve as the bridge that connects applications, enabling us to incorporate social media integration, payment gateways, mapping services, weather data, e-commerce functionalities, email services, and cloud computing capabilities into our own projects with ease.

We've also touched upon different API architectures, such as REST, SOAP, and GraphQL, each with its own strengths and areas of application. Understanding these architectures empowers us to make informed decisions when designing and implementing APIs in our projects.

Hope you gained some valuable insights. Do follow me for more such articles.

Thank you for your time.