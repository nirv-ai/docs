# nirvai docs

documentation for the NIRVai platform

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## THE CASE for EC2

- [its alla bout the benjamins baby](https://www.mobilise.cloud/aws-elastic-kubernetes-service-eks-ec2-vs-fargate/)
- [dont hold your breath](https://github.com/aws/containers-roadmap/issues/622)
- [general fargate spot](https://aws.amazon.com/blogs/aws/aws-fargate-spot-now-generally-available/)
- [cost optimization ecs launch type: farget vs ec2](https://aws.amazon.com/blogs/containers/theoretical-cost-optimization-by-amazon-ecs-launch-type-fargate-vs-ec2/)
- [saving money: one nomad jobspec at a time](https://aws.amazon.com/blogs/containers/saving-money-pod-at-time-with-eks-fargate-and-aws-compute-savings-plans/)
- [savings plans](https://aws.amazon.com/savingsplans/)
- [ec2 spot instance guide](https://www.cloudbolt.io/guide-to-aws-cost-optimization/ec2-spot-instances/)
- [aws data transfer costs](https://www.cloudbolt.io/guide-to-aws-cost-optimization/aws-data-transfer-pricing/)

### general guidance

- For sustained, predictable tasks, a highly utlilized EC2-based launch could help to optimize costs, since the instance type best suited for the required task capacity can be selected at a lower cost than Fargate with the same capacity

```sh
## EC2: m5.8xlarge
## containers: 8
# EC2 utilization: Memory Reservation Rate 6.25%, vCPU Reservation Rate 12.5%
# > Fargate has a 87% saving over an EC2 equivalent
# EC2 is fully utilized
# > EC2 launch type sees over 20% price savings.

```
