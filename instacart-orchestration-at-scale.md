
[Orchestrating at Scale How Instacart Manages 20M+ Daily Workflows](https://youtu.be/ECN57ZB9xRs)<br>
May 29, 2025<br>
Presenter: Anant Agarwal, Staff Software Engineer, Instacart<br>

Orchestration
- an automated coordination and management of workflows
- ensures reliablity and scalability

Workflow
- a fundamental unit of orchestration

![Benefits of an Orchestration Workflow](/images/benefits-of-orchestration-workflow-instacart.png)

![Airflow Deployment at Instacart](/images/airflow-deployment-at-instcart.png)

### Scale
- Fullfilled 300 million orders in FY 2024
  ![Usage by numbers](/images/airflow-temporal-usage-by-numbers-instacart.png)


### Technology Stack

| **Product / Component**  | **Notes**               | **Slides**                      |
|:-------------------------|:------------------------|:--------------------------------|
| [Apache Airflow](https://airflow.apache.org) | • Robust batch processing system |![Apache Airflow](/images/apache-airflow-at-instacart.png) |
| [Temporal](https://temporal.io) | • Horizontal scaling<br> • Low latency<br> • Exactly-once execution guarantee<br> • e.g. payment capture, display ads | ![Temporal](/images/temporal-at-instacart.png) |
| [Postgres](https://www.postgresql.org) with extensions<br> • [pgcat](https://pgxn.org/dist/pgcat)<br> • [pgbouncer](https://www.pgbouncer.org) | backend database | |
| [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/)| Applications deployed as Dockerized containers | |
| [Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html) | Open source distributed task queue system | |
| [Apache Cassandra](https://cassandra.apache.org/) | | |
| [Elasticsearch](https://github.com/elastic/elasticsearch) | | |
| [Terraform](https://developer.hashicorp.com/terraform) | IaC | |
| [Datadog](https://www.datadoghq.com/) | Monitoring & Alerting | |


### Key Takeaways

1. Invest in abstractions and Wrappers<br>
   Build common patterns for everyone to use.

2. For consistent user experience, focus on Infrastructure as Code (IaC)

3. Use Declarative Configs
