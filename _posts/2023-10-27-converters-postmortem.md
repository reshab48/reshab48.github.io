---
layout: post
title: "The High Cost of a Simple Oversight: My Experience with Lambdas"
---

In the vast world of tech, it's often the small things that catch us off guard. A single oversight, a minor lapse in attention, can lead to bigger consequences. I recently had one such experience which, though not catastrophic, turned out to be a valuable lesson for me.

### **The Background**

At the heart of our operations are the converters lambdas. If you're unfamiliar with this term, these are essentially programs that transform websites into PNG and PDF files. Traditionally, the process involved the frontend initiating a call to an asynchronous lambda, which in turn, triggers our primary lambda. This primary one then gets to work: using Puppeteer, it generates files and stores them on an S3 bucket. This design was replicated for both PNG and PDF files.

### **A New Idea**

However, as with all things tech, there's always room for improvement. I had the inspiration to shift our lambdas to an image-based architecture. Instead of initiating the Puppeteer browser process with each request, why not have it run continuously using PM2, a process manager? This means subsequent calls to the lambda could reuse an already 'hot' browser process. It was a promising theory. With our lambdas handling around 40 invocations per minute, even slight efficiencies could lead to significant time savings.

Transitioning to this new model was relatively seamless. And while the speed improvements weren't groundbreaking, they were still noticeable.

### **The Oversight**

But here's where I tripped up. Initially, I intended to have just one lambda for each file type, eliminating the need for the asynchronous trigger. With a single S3 bucket catering to both our staging and production environments, we distinguished between the two using prefixes: `staging/` and `production/`. Due to some unforeseen technical hitches involving SQS events and queue congestion, I reverted to using the asynchronous lambda.

The problem? I inadvertently duplicated the prefixes, leading to paths like `staging/staging/` and `production/production/`.

### **The Fix and Its Price**

To set things right, we had to transfer files from the erroneous prefix to the correct one. The data transfer alone cost us $10. Add to that the developer hours spent in resolving this, and we were looking at an expense of around $250.

While the monetary cost might seem modest, it's important to remember that this could've escalated had it not been for my diligent team, who caught the mistake early. We were looking at around 500,000 files; a number that could've grown exponentially if unchecked.

### **In Conclusion**

In the intricate dance of code and infrastructure, missteps can happen. What's important is to learn from them, stay vigilant, and always, *always* double-check even the 'minor' changes we make. The world of tech is as unforgiving as it is exciting, but it's these experiences that shape us into better professionals.
