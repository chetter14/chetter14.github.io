---
layout: post
title: DevOps. How I set up CI/CD.
---

Yeah, I am back after the long break of posting here anything. I've got a new job that required my full attention and also was dealing with university tasks and labs. But now I kinda broke free from university and have time to share my experience from projects I was working on.

Now I want to tell how I made CI/CD for uni project. The project was to write your web application and integrate CI/CD into it. My teammate made backend (Java) and frontend (TypeScript) - so, all the stuff related to coding. My job was to write CI/CD for the app. The sources of project can be found [here](https://github.com/chetter14/devops).

I made CI/CD pipeline using GitHub Actions. Here I'm going to describe steps that I took in order:

1) Add `build` and `test` jobs. At first you build backend and frontend, and then run unit tests to see whether it fails or not. I haven't encountered anything difficult here.

2) The next thing was to deploy the app in Kubernetes. It was tough because I hadn't work with Kuber before, so I used **Kompose** to convert `docker-compose.yml` to deployment and service configs. All the Kuber configs are placed [here](https://github.com/chetter14/devops/tree/main/kubernetes/configs). Not sure about their amount, maybe it could be done in less number of configs.

We have chosen to deploy application in *Yandex Cloud* because it gives you free access until you hit some CPU / resource usage limit. But it was fine, the project is meant to be educational, no serious load.

After playing with Kubernetes commands like - `kubectl apply -f`, `kubectl top pods`, etc. - I got the general idea and could successfully deploy the application.

3) Add horizontal scaling of backend depending on a load was okay. You just add a separate config - `backend-hpa.yaml` - in which you describe the service you want to scale, how fast, when to scale, etc.

4) Attach Grafana to present all the metrics of application: *ram usage, request counts, average cpu load*, etc. And this info is taken from each pod. I think the [dashboard](https://github.com/chetter14/devops/blob/main/kubernetes/configs/grafana-dashboard.yaml) we used (I got it from my teammate) is pretty good so you can use it in your apps.

And add step of pushing Docker images of backend and frontend to Docker Hub, so that it *could be pulled from Docker Hub during deployment*.

5) Add additional *code analysis in SonarQube* to find out **security errors, various hotspots in code, check code coverage**. If coverage is less than *80 percent* or some issues were found, then the pipeline is going to be halted. This is the default SonarQube Quality Gate which you cannot adjust unless you upgrade your profile. 

The process of SonarQube analysis is also integrated into CI pipeline.

6) And the last step is to deploy your application automatically - *implement CD in the GitHub Actions last job*. I made a bunch of *GitHub Secrets* using which it connects to remote host (that Yandex Cloud provided), copies config files there, and deploys them successfully. 

You can notice *Telegram bot* related variables. It was part of the task to integrate Telegram bot so that your application send logs to this bot. There is nothing captivating for me here, and it doesn't match CI/CD topic.

So, that's how I set up CI/CD in project. It was sometimes interesting, sometimes annoying. Now I better understand why DevOps is a separate job vacancy in companies. It gathers much of your attention and requires not a beginner-level expertise.