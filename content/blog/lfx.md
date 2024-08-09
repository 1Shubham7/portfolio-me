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

Contributing to open source, specially to the Cloud Native Community Foundation (CNCF) has been one of the most effective ways for me to learn programming, understand how large codebases work and directly work unedr the guidance of highly talented engineers in the cloud native development space. It's been an year for me as a contributor in the CNCF ecosystem and now as I wrap up my LFX (Linux Foundation Mentorship) program I want to share my journey in open source - from contributing to CNCF projects to getting into the prestigious LFX (Linux Foundation Mentorship) program and completing it successfully.

## My first contribution to CNCF

I got know about CNCF through YouTube, I heard many engineers in cloud native space talk about CNCF and I went to the CNCF home page but I couldn't understand anything, I think the reasons was that I had never anything similar to that. Is this a company, or is to a project? or is it like a facebook group? But as I leaned more about cloud native technologies like Docker, Kubernetes I understood what CNCF is and what it stands for, by this time I also started searching of some CNCF projects to contribute and one such project was CNCF ORAS (OCI Registries As Service), basically it is a tool for pushing and downloading OCI Artifacts from the OCI Registries. Special thanks to I made my first CNCF contribution in CNCF ORAS .


## Applying for LFX for the first time

I started with changes in readme for the ORAS project and slowly tried understanding the project, watching tutorials, going through docs and trying it locally. I then started contributing to the docs and then made my first code contribution in ORAS. I contributed to ORAS for a month and I wanted to apply for LFX with them but they did not had a proposal for that term so I had to pick up a different project. Later I realized I need to improve my skills before applying to LFX so I skipped that term and rather took an internship offer I had from a startup.

For next few months all my focus was on building projects, learning go and Kubernetes, and working on some internships. If we go back a little bit, when ORAS decided to not come for LFX I started searching for other CNCF projects I that is when I saw KubeEdge, the project I am currently working under and Kyverno where I became an official contributor. So after all this was going I started learning about Kyverno, one of my friends recommended me to contribute to Kyverno and I would also recommend any newbie who wants to start contributing that choose a project that is active, has a really good documentation and maintainers a supportive. Luckily all three projects I have contributed to till now have been amazing and specialy at KubeEdge, we are continuously trying to improve our documentation so if you think you would want to help us, please join the community and contribute.

Tip - Kyverno is also a great project if you are decent at go + Kubernetes since it has very well structured documentation, very active community and helpful maintainers. If you want to start contributing to Kyverno my recommendation would be to start by watching some CNCF talks about what Kyverno is and what it does, then complete the free certification from the nirmata website and then pick up any good first issue from the project (they are very active and frequently post up good first issues to solve).

Even I did the same, I also went through some CNCF talks about Kyverno, completed the certification and then started contributing. I started with some code contributions and then also created a section in their documentation about admission controllers which may go live after some time with changes form the maintainers.

I contributed to Kyverno for a few months while also managing my internship and college classes and finally the time came when projects for LFX term 2 2024 were released. I saw the list of CNCF projects that had applied for term 2, 2024 and Kyverno was not there and once again I had to change my project in the last minute. I was going through all the projects that were coming in LFX, I saw KubeEdge there, I had some idea about what KubeEdge is and I found the projects that KubeEdge had for LFX term 2 are doable for me. My focus was on two projects, one was about documentation improvement and the other one was about tests enhancement. While I was more interested in code contributions but I had a background of working as a technical writer for 8 months and I also had good contributions in Kyverno documentation so I decided to focus more on the documentation enhancement project and alddo give time to the tests enhancement project.

For the documentation improvement project, I went through all the tutorials present online for KubeEdge, I went through the entire documentation and I tried improving the documenation by bringing in minor changes in sentence refactoring, rephrasing docs and some other minor improvements. majorly, I wrote some blogs for different KubeEdge releases from v1.10 to v1.17, for this I used the changelogs of these releases as reference. While for the tests enhancement project, I started learning about writeing unit tests, E2E tests using ginkgo and gomega, TDD (Test driven development) and tinkering with some libraries I could use for testing in golang. I had about a month to contribute to KubeEdge, understand the project and prepare two cover letter for the two projects. By the time LFX registrations started, I had good number of Pull Requests merged in KubeEdge (11) and some were being reviewed.

After this the tasks I had to complete in order to complete my LFX application were - submit a resume, and submit a cover letter (which is the case for most of the projects, some projects can also ask for another task testing your knowledge about the tool). I already had a good resume prepared so I made minor changes in it, I started preparing my cover letter for both the projects, in the cover letter I mentioned my past contributions to the CNCF community as well as to Kyverno, other than that I answered honestly all the standard questions that LFX asks and presented how I would go about working on the project for three months.

With the past experience I had as an technical writer, and contributions I had to the KubeEdge documentation I thought I had more chances of getting in to the "documentation improvement" projects. While I would have loved to work under any of the two proejcts and I just wanted to get in into LFX, on the result date I was surprised for good that I have been selected to the "tests enhancement" project. I was super excited to work under my mentors and I just got started by revising some stuff related to unit testing and E2E testing, and then just direclty jumped into writing tests.

For the first few weeks I worked on writing Unit tests for modules where I didn't see much tests, I was that at most of the places the project used standard libraries for testing while an assertion library like testify/assert could be a better choice. So I started with discussing this with my mentor, I discussed about the advantages of using an third party assertion library and my mentor agreed and hence for couple of weeks I wrote a lot of unit tests for a lot of modules and for the modules where already tests were written, I migrated those from using standard library to testify/assert.

Since I was assuming that I would be seleceted into the documentation improvement project, before LFX when I could have learned more about writing tests, I was busy contributing to documentation, learning more about the project and watching tutorials on how to write better documentation. Because of this my early weeks at LFX were qiute hectic, I was not only writing tests but also learning how to write E2E tests, figuring out how to use ginkgo and gomega and looking around if there were better alternatives for E2E testing than ginkgo and gomega. Since all I had to do was writing tests, my LFX mentorship went quite smooth (work wise), I wrote unit tests till the end of the mentorship because the project had very less test coverage (about 40%) and the goal of the community was to bring it to 80%, while the mentors did not burden me with this goal. They told me to improve the coverage as much as I could.

![LFX Acceptance Mail](/blog/LFXblog/lfx-mail.png)

![Kyverno Logo](/blog/LFXblog/kyverno-img.png)


## Applying for LFX for the third time

## An finally I got in!

## My LFX mentorship Experience

## Final thoughts

## A small thank you note to Linux Foundation and CNCF

## A small thank you note to my mentors - Fiexer Xu and Bao Yue