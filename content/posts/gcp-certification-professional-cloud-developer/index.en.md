---
title: "Google Cloud Professional Cloud Developer Certification"
date: 2022-07-11
translationKey: "posts/gcp-certification-professional-cloud-developer"
categories: 
  - gcp
image: "../../images/gcp.webp"
tags:
  - gcp
  - certification
  - cloud
---

I have now obtained the [Professional Cloud Developer](https://cloud.google.com/certification/cloud-developer) qualification. I didn't get any qualifications last year, so this will be my first qualification in a while. Up until now, I have mainly used AWS, so I thought it would be a good idea to obtain a related qualification, but I am taking the exam because I want to know more about GCP services and how it differs from the AWS qualification.

As was the case last time, I was able to successfully obtain the qualification, so in this post I would like to briefly talk about how I prepared for the exam.

## what is the exam like

I have previously written an article about [AWS certifications](../aws-certification-associate-developer/), but even if the provider is different, the fundamentals of cloud services remain the same, so the exam felt similar. Basically, I felt that preparation would not be too difficult if you already had experience with concepts such as IaaS and PaaS, and with deploying and configuring backend services in the cloud.

There are two options for taking the test: remote monitoring and on-site monitoring. In my case, on-site monitoring was only available on weekdays, so I chose remote monitoring. As with the AWS exam, remote monitoring requires you to properly prepare the room where you will take the exam. What you need to prepare is clearly specified on the [KRYTERION site](https://kryterion.force.com/support/s/article/Launching-your-Online-exam?language=en_US), so it is better to check it in advance. Personally, I found it quite difficult to organize the room to meet this requirement, so I thought on-site monitoring would be easier.

Remote monitoring tests are conducted using a dedicated browser provided by KRYTERION. When the browser starts, other apps are minimized, and if you have an external monitor connected, that monitor's screen turns black. In my case, I was using an Apple Silicon Mac, so I was more concerned about compatibility, but I was able to run it without any problems. They also provide a [system check](https://www.kryteriononline.com/systemcheck/#) website, so you may want to check there to see if you can take the exam on your own PC. You also need to register biometric information, but that is simply a matter of taking a photo. On the day of the exam, you have to confirm your registered biometrics before the exam begins, and go through simple procedures such as showing the room and presenting your ID under the instructions of the invigilator.

In the case of other qualifications, I think there were many cases where you could get the results on the spot or on the same day, but with the GCP qualification, it takes about a week for the certification email to arrive. Considering the fact that the perfect score and score are not known, I believe that various situations at the time of the exam are taken into account when making a comprehensive judgment. Since you are supposed to use a dedicated browser for the exam, I think that all your keystrokes and camera footage were probably recorded as well.

## how did you prepare

First, I searched for mock exams on Udemy. There are English versions and Japanese versions, but as stated in the reviews, most of the questions are easier than the actual test. In my case, I took the 5-part English exam 4 times and got over 80% of the answers on all of them, but in the real exam, there were many more difficult questions (things that weren't asked in the mock exam or were more complex), so I didn't think this was enough.

There is also a mock test called [the official one](https://docs.google.com/forms/d/e/1FAIpQLSc_67KaPnNwQrLZ7kuhw-aubz7gMAwY6DQwRJYcW0qlG-iajA/viewform), which has many questions that are relatively similar to the actual test (some questions are deep, but some are almost exactly the same), so I think it would be better to study it while checking the documentation for each service that appears in the questions.

In addition to the mock exams, there was information that there were many questions on Kubernetes, and since I had never actually used it, I did a lot of research on that. Other than that, I just checked once the concepts of testing and deployment (Blue-Green, Canary, etc.).

## What are you asked?

It is better to prepare by referring to the [HipLocal case study](https://cloud.google.com/certification/guides/cloud-developer/hip-local-case-study?hl=ja). Although the number of questions related to HipLocal itself is not high overall, there are various question patterns using this case study beyond those in the mock exam, so I think it is a good idea to study the migration flow before the exam by comparing it with GCP services.

One, you don't need to remember all the HipLocal case studies. Personally, I was quite concerned about this, but in the actual test, a case study was presented on the right side of the browser, so I was able to refer to it and solve the problem. The content displayed in the browser states that there is a possibility that case studies other than HipLocal may be presented, but I feel that this is because other tests have similar problems.

You can find the scope and details of the questions asked on other blogs, and I don't remember the details, so I won't go into details, but I feel like there were a lot of questions related to Kubernetes, followed by storage and serverless issues. Another thing that left an impression on me was that there were issues that mentioned [Anthos](https://cloud.google.com/anthos) and [istio](https://istio.io/). I had only heard the names of both of them, so I didn't even know if I had answered the question correctly.

## After getting accepted

I received an email with a code to receive benefits from the Google Cloud Certification Perks Webstore. The types of benefits you can receive seem to change from time to time, and other blogs have written that you can receive items such as a Bluetooth Speaker, but in my case I had to choose from the following.

![Google Cloud Certification Perks Webstore](perk.webp)

I wanted to choose the zipper hood, but this time I could only choose the size 2XL, so this time I chose the one without the zipper. If you don't have something in particular that you want, there is an option to donate, so you may want to choose that option.

You can check the certification of qualifications from the site [Accredible](https://www.accredible.com/). This site has the ability to link certifications to the site and share them on Linkedin, as you can see with [Credly](https://www.credly.com/) for AWS and Oracle qualifications. The certificate can also be downloaded as a PDF.

First, in the case of AWS, the expiration date of the qualification was 3 years, but in the case of GCP it was 2 years. In the cloud, service specifications often change, and new services are often introduced, so I think it's natural to have an expiration date, but considering the amount of study and the exam fee ($200), it feels a little short.

## lastly

After taking the exam, I was worried that I might have failed this time because there were so many questions that were so different from the mock exams, and even though I later received an email saying I had passed, I still didn't really feel it. However, I feel like that's because I only memorized the questions asked in the mock exam.

In some actual questions, you can find the correct answer by carefully reading the question and the options, and in many cases you can verify it by checking the GCP documentation. So I recommend reading the documentation for each service first, especially the content in the official [certification guide](https://cloud.google.com/certification/guides/cloud-developer). Mock exams are only mock exams, so even if you get a good score, letting your guard down can still lead to failure.

Regarding the difficulty level of the questions, I inevitably compared it to the AWS certification I had previously obtained, but since that was an associate level certification, I felt that this certification was more difficult. Personally, even after passing the exam, I felt like I needed to study more to truly call myself a professional. However, if you use various GCP services on a regular basis and have experience migrating from on-premises, it may not be that difficult, so I think you're well qualified to pass if you don't let your guard down.

See you soon!
