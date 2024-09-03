---
title: "My journey from a CNCF contributor to LFX mentee"
description: ""
dateString: August 2024
draft: false
tags: ["Kubernetes", "KubeEdge", "Golang", "E2E Testing", "Ginkgo", "Gomega", "Testify", "Behavior Driven Development (BDD)", "Test Driven Development (TDD)"]
weight: 98
cover:
    image: "experience/lfx.webp"
---

Contributing to open source, specially to the Cloud Native Community Foundation (CNCF) has been one of the most effective ways for me to learn programming, understand how large codebases work and work with experienced engineers on large scale projects. It's been a year for me as a contributor in the CNCF ecosystem and I also have recently completed my LFX (Linux Foundation Mentorship) program as a mentee. As I wrap up my LFX  program I want to share my journey in open source - from contributing to CNCF projects to getting into the prestigious LFX (Linux Foundation Mentorship) program and completing it successfully.

## My first contribution to CNCF

![ORAS Logo](/blog/LFXblog/oras.png)

Let's start it from how I got introduced to CNCF, I got know about CNCF through YouTube videos, I saw some YouTubers in cloud native space talk about CNCF and I went to the CNCF home page but I couldn't understand anything, I think the reason was that I had never anything similar to CNCF. I was confused whether CNCF is a company where I can get placed? or is it a project? or is it like a community? But as I leaned more about cloud native technologies like Docker, Kubernetes I understood what CNCF is and what it stands for, by this time I also started searching of some CNCF projects to contribute to and one such project was [CNCF ORAS](https://oras.land/) (OCI Registries As Service), basically it is a tool for pushing and downloading OCI Artifacts from the OCI Registries. I contributed to ORAS for a month where I learned the how to contribute to a CNCF project. And with my ORAS experience I can suggest a newbie in open source that the best way to make your first contribution is to first, understand what the project is by reading docs and try using it, then using docs set it up locally and then look up for issues labelled "good first" or "help wanted" on their GitHub repo and start by solving those issues. 

The first contribution I made to ORAS was some basic changes in the readme file, and then I tried understanding the project, watching tutorials on the [CNCF YouTube channel](https://www.youtube.com/@cncf), going through docs and trying to set it up locally. I then started contributing to the docs and then made my first code contribution in ORAS.

## Applying for LFX for the first time

I contributed to ORAS for a month and I wanted to apply for LFX with them (this is during LFX 2023 term 3) but they did not have a proposal for that term so I had to pick up a different project. Later I realized I need to improve my skills before applying to LFX so I skipped that term and rather I went forward with an internship offer I had from a startup.

## Contributing to CNCF Kyverno

![Kyverno Logo](/blog/LFXblog/kyverno-img.png)

For next few months along with my internship I was focusing was on building projects, learning go and Kubernetes, and leaning about another CNCF project called [CNCF Kyverno](https://kyverno.io/), it is a Kubernetes native policy engine. If we go back a little bit, when ORAS did not come for LFX, I started searching for other CNCF projects I that is when I saw KubeEdge, the project I am currently working under and Kyverno where I became an official contributor later. I started learning about Kyverno, one of my friends recommended me to contribute to Kyverno and I would also recommend any newbie who wants to start contributing to open source that you should choose an active project (where maintainers are active and regularly help contributors). I can recommend Kyverno to any beginner because it has a really good documentation and maintainers a very supportive. Luckily all three projects I have contributed to till now have been amazing. Also I would suggest that at KubeEdge, we are continuously trying to improve our documentation so if you think you would want to help us, please join the community and contribute.

**Tip** - If you want to start contributing to Kyverno my recommendation would be to start by watching some CNCF talks about what Kyverno is and what it does, then complete the free certification from the [Nirmata website](https://learn.nirmata.com/) and then pick up any good first issue from the project (they are very active and frequently post up good first issues to solve).

Even I did the same, I also went through some CNCF talks about Kyverno, completed the certification and then started contributing. I started with some code contributions and then also created a section in their documentation about admission controllers.

## Applying for LFX again

![KubeEdge Logo](/blog/LFXblog/kubeedge.png)

I contributed to Kyverno for a few months while also managing my internship and college classes and finally the time came when projects for LFX term 2 2024 were announced. I saw the list and Kyverno was not there and once again I had to change my project just before a month from LFX. I was going through all the projects that were coming in LFX, I saw [KubeEdge](https://kubeedge.io/) there, KubeEdge is a Kubernetes native edge computing framework. I had some idea about what KubeEdge is and I found the projects really interesting so I decided to apply for LFX under KubeEdge. I applied for two projects, one was about writing new documentation and the other one was about test enhancement. While I was more interested in code contributions but I had a background of working as a technical writer for 8 months and I also had good contributions in Kyverno documentation so I decided to focus more on the documentation enhancement project and also give time to learn things for the tests enhancement project.

For the documentation improvement project, I went through all the tutorials present online for KubeEdge, I went through the entire documentation and I tried improving the documentation by bringing in changes in sentence refactoring, rephrasing docs and some other improvements. majorly, I wrote some blogs for different KubeEdge releases from v1.10 to v1.17, for this I used the changelogs of these releases as reference. While for the tests enhancement project, I started learning about writing unit tests, E2E tests using ginkgo and gomega, TDD (Test driven development) and tinkering with some libraries I could use for testing in golang. I had about a month to contribute to KubeEdge, understand the project and prepare two cover letter for the two projects. By the time LFX registrations started, I had good number of Pull Requests merged in KubeEdge (11) and some were being reviewed.

After this, the tasks I had to complete in order to complete my LFX application were - submit a resume, and submit a cover letter (which is the case for most of the projects, some projects can also ask for another task testing your knowledge about the tool). I already had a good resume prepared so I made minor changes in it, I started preparing my cover letter for both the projects, in the cover letter I mentioned my past contributions to the CNCF community as well as to Kyverno, other than that I answered honestly all the standard questions that are asked, other that this I also mentioned how I would go about working on the project for the three months.

## And I finally got in!

With my past experience as a technical writer, and the contributions I made to the KubeEdge documentation as well as Kyverno documentation I thought I had more chances of getting in to the documentation project, on the result date I was surprised for good that I have been selected to the "tests enhancement" project, this was even better because now I would get to make code contributions. I was super excited to work under my mentors. I just got started by revising some stuff related to unit testing and E2E testing, and then jumped into writing tests.

![LFX Acceptance Mail](/blog/LFXblog/lfx-mail.png)

## My LFX mentorship Experience

For the first few weeks I worked on writing Unit tests for modules where I didn't see much tests, I was that at most of the places the project used standard libraries for testing while an assertion library like testify/assert could be a better choice. So I started with discussing this with my mentor, I discussed about the advantages of using an third party assertion library and my mentor agreed and hence for couple of weeks I wrote a lot of unit tests for a lot of modules and for the modules where already tests were written, I migrated those from using standard library to testify/assert.

Since I was assuming that I would be seleceted into the documentation improvement project, before LFX when I could have learned more about writing tests, I was busy contributing to documentation, learning more about the project and watching tutorials on how to write better documentation. Because of this my early weeks at LFX were qiute hectic, I was not only writing tests but also learning how to write E2E tests, figuring out how to use ginkgo and gomega and looking around if there were better alternatives for E2E testing than ginkgo and gomega. Since all I had to do was writing tests, my LFX mentorship went quite smooth (work wise), I wrote unit tests till the end of the mentorship because the project had very less test coverage (about 40%) and the goal of the community was to bring it to 80%, while the mentors did not burden me with this goal. They told me to improve the coverage as much as I could.

## Final thoughts

The 2.5 months of LFX mentorship was the best learning experience I had in tech. In these 2.5 months, I explored how to use KubeEdge ,worked with my mentors to improve test coverage of the project as well as contribute to the documentation. I am really greateful that I got to work on an emerging CNCF project with such experienced engineers as my mentors.  Working closing with such great engineers have taught me a lot about cloud native application development, testing and a lot more stuff. I am thankful to the Linux Foundation for conducting such an amazing open source program and doing it consistently over the years.

## A small thank you note to Linux Foundation and CNCF

I also want to thank Sriji amarthan, sadfa, asd and everybody in the Linux Foundation team. During this LFX period I also went through a cyber fraud and the people their were kind to let me refill my stipend report and patiently waited for the whole thing to fix. Thank you guys, you guys are awesome :)

## A small thank you note to my mentors - Fiexer Xu and Bao Yue

I would also like to thank my mentors Feixer Xu and Bao Yue for being so helpful and so supportive throughout the mentorship. I am thankful that you gave me some time from your busy days and helped me out wherever I got stuck.