---
layout: post
title:  "The Ultimate CKAD Prep Guide"
author: israel
categories: [ 'Cloud Native' ]
tags: [ containers, ckad, kubernetes certification, application, developer, cloud-native ]
image: https://user-images.githubusercontent.com/2548160/104125779-ea666400-5350-11eb-8986-333a1ff5662e.png
date:   2021-01-10 15:01:35 +0300
description: "The Certified Kubernetes Application Developer (CKAD) program was developed by the Cloud Native Computing Foundation (CNCF). I passed the exam and these notes summarise what you should expect if you’re preparing it." 
#permalink: /blog/ckad-prep-guide.html
---

The Certified Kubernetes Application Developer (CKAD) program has been developed by the Cloud Native Computing Foundation (CNCF), in collaboration with The Linux Foundation, to help expand the Kubernetes ecosystem through standardized training and certification.

I passed the (CKAD) Exam in June 2020. These notes summarise what you should expect if you’re preparing for the CKAD exam.
### Preparation

CKAD is a hands-on performance-based exam. As a result, lots of practice to build enough muscle memory over an extended period is an absolute requirement.

> Beware of the “curse of knowledge”.

Whilst your mileage may differ if you are already familiar with Kubernetes and perhaps you work with it every day, be cautious of the “[course of knowledge](https://en.wikipedia.org/wiki/Curse_of_knowledge)”. Your prior knowledge, if not properly channelled can trip you. I know folks who know Kubernetes more than I do but failed the exam. You need muscle memory to help you gain speed and accuracy.

<p class="aligncenter">
<img class="lazyimg" src="https://miro.medium.com/max/500/1*l9za84HLq2ISOrsRAwkghw.png" width="700" height="550"/>
</p>
These are the materials that I used:

1\. [Kubernetes Certified Application Developer (CKAD) with Tests](https://www.udemy.com/course/certified-kubernetes-application-developer/) — Fantastic Udemy course, especially if you’re new to Kubernetes. I would recommend that you start with this course.

2\. [CKAD Exercises](https://github.com/dgkanatsios/CKAD-exercises/) — I got bored of watching Udemy videos mid-way into the course, this is not a reflection on the quality of the content, it’s due to my personality and the way I learn. I like to get stuck in. I wanted to dig deeper the moment I felt comfortable with the general concept; my quest led me to [CKDA Exercises](https://github.com/dgkanatsios/CKAD-exercises/). I went over the whole exercises (the _number of times is withheld so you can do it at your pace_) until I could answer ALL the questions without looking at the answers or making reference to the documentation, except for the persistent volume and network policy questions. I was also lucky enough to able to contribute to the CKAD Exercises repo.

> ‘CKAD Exercises’ is not enough though, seriously!

![CKAD Cert](https://cdn-images-1.medium.com/max/600/1*10Rw_8fLdUefu4Q6K_f9ww.png)

Most blog posts that I read before the exam gave me the impression that being able to answer all the questions in CKAD Exercises is an indication that one is ready for the exam. Whilst this assertion may be true for older versions of the exam, I found it not to be true in my experience with version 1.18. The exercises provide a good foundation, but the actual exam questions are a lot harder and are not as straightforward. I felt the need for more time-bound practices so I went back to the [KodeKloud course](https://kodekloud.com/p/kubernetes-certification-course-labs) via Udemy but this time, I focused ONLY on the mock exams and the lightning labs. You may skip the ReplicaSets and Ingress Controller questions as it’s not relevant to the exam (anymore). Stick to the [curriculum](https://github.com/cncf/curriculum).

3\. CKAD Simulator — My preparation took a bit longer because the exams were upgraded from Kubernetes 1.16 to 1.18 when I was almost ready to sit for it, I had to add about 3 more weeks to practice on 1.18 to unlearn the deprecated commands and to learn the new 1.18 imperative commands. The contents on [killer.sh](https://killer.sh/ckad) simulator is excellent, especially with the new scoring system. The exam environment and layout is kind of similar to the actual exam. Aim to score ≥ 60% and pay close attention to the network policy question — note the difference in the way ports are defined as an OR or an AND operator.

4\. Linux Academy — My crusade for more practice led me to the [Linux Academy CKAD](https://linuxacademy.com/course/certified-kubernetes-application-developer-ckad/) course. At first, I was disappointed as the test environment is in version 1.13 (at the time of writing this note), the tutor solved most of the questions by writing YAML too; don’t do this in the exam, use imperative commands wherever possible. Having said that and without going into too much detail (for obvious reasons), I attribute my success to the ‘quality’ of Linux academy’s CKAD practice questions. Take all the CKAD practice questions in Linux academy very seriously.

> Take all the CKAD practice questions in Linux academy very seriously

I didn’t go through any of the video tutorials on Linux Academy, but what I can say is focus on each session’s practice questions about 2–3 days before the exam.

5\. Kubernetes.io docs — Go through the [task session](https://kubernetes.io/docs/tasks/) of the docs.

### Exam Tips

1.  Kubernetes.io search: I got a bit frustrated by typing long words like “persistentvolume claim” or “networkpolicy” into the search box, so one day I thought I’d try the `kubectl` shortcodes instead. Guess what? It worked! So instead of typing “persistentvolume claim” or “network policy” into the docs, I simply typed in `pvc`, `netpol`, `pv`, etc and I got the same result set. Do all you possibly can to save time.
2.  **VIM**: Get comfortable with VIM text editor. Please refer to the “[CKAD browser terminal](https://codeburst.io/the-ckad-browser-terminal-10fab2e8122e)” guide.
3.  **Use AutoComplete:** Autocomplete is enabled for you in the exam, but it will not work if you have the `k=kubectl` alias set. One of the first things I did in the exam was to add `complete -F __start_kubectl k` command to `~/.bashrc`. I memorized the command, but you don’t have to as you can copy it from the [docs](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#bash). Autocomplete is an absolute time-saver. For example, if I need to quickly delete a pod called webone, I’d type: `k del_<tab-key>_ po we_<tab-key>_ --for_<tab-key>_ --gr_<tab-key>_ 0.` This translates to `k delete po webone --force —-grace-period 0`
4.  **Persistent Volumes:** Practice this [Persistent Volume question](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) thoroughly. I will, however, recommend that you practice this on a cluster with 2 or more worker nodes for the following reasons: `i)` SSH into any of the nodes and create the said file `ii)` You need to `sudo -i` to be able to create files in some directories such /etc/, /opt/ etc. `iii)` Type `exit` to return to the master node `iv)` Ensure that the nginx pod is scheduled on the intended node by using either nodeName (this is faster), nodeSelector or nodeAffinity.
5.  **Always return:** You’d need to use `sudo -i` to be able to create or edit files in certain directories, remember to type `exit` when done. Always remember to return to the master node.
6.  **Use** `**~/.bashrc:**` Due to 4 and 5 above, do not simply type your aliases in the terminal, use `~/.bashrc` for all your aliases so you don’t have to do it all over again when you return to the master node. My `~/.bashrc` aliases in the exam look like this:

```cpp

alias k=kubectl  
alias kns='k config set-context --current --namespace  ' 
alias kgc='k get po -l x=y'                                 
complete -F __start_kubectl k  
dr='--dry-run=client -o yaml'

```

Don’t forget to reload your terminal by typing `source ~/.bashrc`

I bet you’re familiar with all of the above aliases, except you’re questioning why I’d need to do alias `kgc=’k get po -l x=y’` instead of `kgc=‘k config get-contexts’`? Your answer is in tip #7.

My `~/.vimrc` settings look like this:

set tabstop=2  
set expandtab   
set shiftwidth=2 # very useful for indentation

7\. **Less is more**: Except it’s a one-liner solution, I did not use `-n <namespace>` in any questions in the exam because I sometimes forget to add `-n`. I used `kns <namespace>` to be absolutely sure. `k get po -l x=y` on the other hand, returns a decluttered name of the namespace to me. For example:

`~ $ k get po -l x=y`

No resources found in **_israelo_** namespace.

Now, go ahead and type `k config get-contexts` , the result is a lot if all you’re interested in is just the name of your current namespace.

![Cert](https://cdn-images-1.medium.com/max/1200/1*qY6iMu4ScDNarozEiP-YTQ.png)

8\. **Namespaces**: When you switch to a new cluster, notice that the namespace doesn’t change automatically to the `default` namespace. **_This is so important._** So be sure to return to the default namespace if the question doesn't specify a namespace. `kns default` then `kgc` for your own peace of mind.

9\. **Deployment Strategy**: There is no mention of [deployment strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) in any of the practice resources I mentioned above, including CKAD Exercises. Learn the two types of deployment strategies — Recreate and Rolling update. Know how to change `maxUnavailable` and `maxSurge` values. Practice how to avoid downtime by combining rolling update with `readinesssProbe` and `livenessProbe`

10\. **Immutable objects**: Pods are immutable objects so before you delete a pod, remember to take a backup (`k get po webone -o yaml > 1.pod.yaml`). This also applies to any YAML file that you’re asked to modify. Take a backup before you edit it.

In summary, there’s no such thing as over-preparation in my opinion, the exam is hard, no doubt. Use practice environments that can vet and score your performance under time pressure — good examples are killer.sh and KodeKloud.

I attempted 17 out of the 19 questions and I scored 82%.

Please post a comment below if you have any questions.

Good luck and stay hungry!

<i class="aligncenter"> Notice : This blog was first published on [medium](https://iogbole.medium.com/ckad-1-18-exam-preparation-notes-6c91f8140393) </i>

<p class="aligncenter">
<img class="lazyimg" src="https://user-images.githubusercontent.com/2548160/104130796-b730ce80-536a-11eb-97e8-79b6d2339726.jpg"/>
</p>
