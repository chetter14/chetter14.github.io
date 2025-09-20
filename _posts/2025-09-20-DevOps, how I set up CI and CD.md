---
layout: post
title: DevOps. How I set up CI/CD.
---

Yeah, I'm back after a long break from posting anything here. I got a new job that required my full attention, and I was also dealing with university tasks and labs. But now I'm kinda free from university and have time to share my experience from the projects I was working on.

I want to talk about how I set up CI/CD for a uni project. The goal was to write a web application and integrate CI/CD into it. My teammate built the backend (Java) and frontend (TypeScript) - so, all the coding-related stuff. My job was to write the CI/CD pipeline for the app. The project sources can be found [here](https://github.com/chetter14/devops).

I built the CI/CD pipeline using GitHub Actions. Here are the steps I took, in order:

1) **Add `build` and `test` jobs**. First, you build the backend and frontend, and then run unit tests to see if anything fails. I didn't encounter any major difficulties here.

2) **Deploy the app to Kubernetes**. This was tough because I hadn't worked with Kubernetes before, so I used **Kompose** to convert `docker-compose.yml` into deployment and service configs. All the K8s configs are located [here](https://github.com/chetter14/devops/tree/main/kubernetes/configs). I'm not sure about the number of files; maybe it could have been done with fewer.

We chose to deploy the application on *Yandex Cloud* because it offers free access until you hit certain CPU/resource limits. This was fine for our needs, as the project was meant to be educational and wouldn't handle serious load.

After playing with Kubernetes commands like `kubectl apply -f` and `kubectl top pods`, I got the general idea and was able to deploy the application successfully.

3) **Add horizontal scaling for the backend based on load**. This was straightforward. You just add a separate config — `backend-hpa.yaml` — where you describe which service to scale, how quickly, under what conditions, etc.

4) **Connect Grafana to display application metrics**: *RAM usage, request counts, average CPU load*, etc. This data is collected from each pod. I think the [dashboard](https://github.com/chetter14/devops/blob/main/kubernetes/configs/grafana-dashboard.yaml) we used (which I got from my teammate) is pretty good, so you can use it in your own apps.

I also added a step to push Docker images for the backend and frontend to Docker Hub, so they *could be pulled from there during deployment*.

5) **Add code analysis with SonarQube** to find **security errors, code hotspots, and check code coverage**. If coverage is less than *80 percent* or critical issues are found, the pipeline fails. This uses the default SonarQube Quality Gate, which can't be adjusted without upgrading your plan.

The SonarQube analysis process is also integrated into the CI pipeline.

6) **The final step was to implement continuous deployment (CD)** by adding an automatic deployment job in GitHub Actions. I created a set of *GitHub Secrets* that the pipeline uses to connect to the remote host (provided by Yandex Cloud), copy the config files, and deploy everything successfully.

You might notice *Telegram bot* related variables. Integrating a Telegram bot for sending application logs was part of the task requirements. It wasn't particularly captivating for me, and it doesn't really fit the CI/CD topic.

So, that's how I set up CI/CD for the project. It was sometimes interesting, sometimes annoying. Now I understand better why DevOps is a separate role in companies. It demands a lot of attention and requires more than just beginner-level expertise.