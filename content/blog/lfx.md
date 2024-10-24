---
title: "My journey from a CNCF contributor to LFX mentee"
description: ""
dateString: August 2024
draft: false
tags: ["Kubernetes", "KubeEdge", "Golang", "E2E Testing", "Ginkgo", "Gomega", "Testify", "Behavior Driven Development (BDD)", "Test Driven Development (TDD)"]
weight: 10
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

![LFX Acceptance Mail](/blog/LFXblog/lfx-mail.png)

With my past experience as a technical writer, and the contributions I made to the KubeEdge documentation as well as Kyverno documentation I thought I had more chances of getting in to the documentation project, on the result date I was surprised for good that I have been selected to the "tests enhancement" project, this was even better because now I would get to make code contributions. I was super excited to work under my mentors. I just got started by revising some stuff related to unit testing and E2E testing, and then jumped into writing tests.

## My LFX mentorship Experience

### The first half of LFX mentorship

After I received the acceptance mail, I was really excited to get started with my mentorship, I communicated with my mentors and they told me to start by writing unit tests for simple files since those are more straight forward. This was the first time I was writing unit tests so I started by watching tutorials and deciding which library to use for testing. In KubeEdge we were using the standard testing library for unit tests and if else statements for assertion. I researched a bit and saw that using an assertion library like `testify/assert` would make more sense as it removes much boilerplate from the tests, and improves speed and quality of understanding and modifying the tests. And that there are negligible downsides to using this library. I talked with my mentor regarding this and we decided to go forward with `testify/assert`.

For these 3 months I was writing unit tests continuously but also working on other tests side by side, in the first week other than writing tests and researching I also wrote a release blog for KubeEdge v1.10. For the next few weeks I was only writing unit tests. I started with simpler and more straight-forward tests and then moved to tests that require mocking and tests that would test the integration (calling) of multiple functions.

I had never written E2E tests before this so I also had to watch tutorials and read documentation for E2E. Our project was using `Ginkgo` and `Gomega` for E2E so I learned about them, went through the official documentation for `Ginkgo` and wrote some E2E tests for dummy projects for practicing. Before the first evaluation (which happens after 1.5 months) I had improved the test coverage of the project by about 5%, written E2E tests for adding the liveness probe configurations in deployment and checking if it works in the edge, and written a few blogs for some KubeEdge releases (v1.10 ~ v1.17). And when I got my mid term evaluations, I had successfully passed them.

![LFX Mid term evaluations](/blog/LFXblog/one-half.png)

### The second half of LFX mentorship

In the second half of my LFX mentorship, I continued writing unit tests, the only difference being they were getting more complex now and I regularly had to visit Kubernetes docs or [https://pkg.go.dev/](https://pkg.go.dev/) for reference. During this time, I wrote E2E tests for verifying that the applications configured with probe works fine on the edge. Other than that since many tests in the project were simply using standard testing library without any assertion library, I migrated those tests to standard library + `testify/assert`.

By the time my LFX mentorship period ended, I had improved test coverage of out project by 10%, written about 20k lines of tests, migrated tests from standard testing library to a new assertion library and written blogs of 7 KubeEdge releases (v1.10 ~ v1.17). I also had my end semester exams during the second half of the mentorship so for a few days became very hectic but overall the mentorship went quite smooth and I got to learned a lot of things. Whenever I needed any help I communicated with my mentors and they were very supportive and helped me wherever I needed them. And with this on 29th August 2024 I received my final evaluations and I had successfully completed my LFX mentorship.

![LFX Mid term evaluations](/blog/LFXblog/two-half.png)

![LFX completion letter](/blog/LFXblog/result.png)

## Final thoughts

The 3 months of LFX mentorship was one of the best learning experience I ever had in tech. In these 3 months, I explored how to use KubeEdge, worked with my mentors to improve test coverage of the project as well as contribute to the documentation. I am really grateful that I got to work on an emerging CNCF project with such experienced engineers as my mentors. Working closing with such great engineers have taught me a lot about cloud native application development, testing and a lot more stuff. I am thankful to the Linux Foundation for conducting such an amazing open source program and doing it consistently over the years and I highly recommend it to everyone that at least applying for LFX if you are a student or somebody interested in FOSS, even the application process and contributions will teach you a lot about FOSS development

### A small thank you note to Linux Foundation

I want to thank the entire Linux Foundation team, specially Nate Waddington and Sriji Ammanath. During this LFX period I also suffered  a cyber fraud and the LF team specially Nate and Sriji were kind enough to pause my payments and patiently waited for the whole thing to be fixed. Thank you guys :)

### A small thank you note to my mentors

I would also like to thank my mentors Fisher Xu and Shelley Bao Yue for being so helpful and so supportive throughout the mentorship. I am thankful that you gave me some time from your busy days and helped me out wherever I got stuck. Thank you mentors, you guys are awesome :)