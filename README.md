# Terraform Associate 004 Pocket Guide

A concise, structured study companion for the **HashiCorp Certified: Terraform Associate (004)** exam.  
This guide is based on the **Official Terraform Documentation** and the [**Terraform Associate 004 Learning Path**](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004) provided by HashiCorp.


## 💡 Why This Project?

I created this guide because I know that not everyone has the budget for paid courses. Official documentation is the gold standard for learning, but it can feel overwhelming when you're just starting out.

I passed my certification using only the official docs by focusing on the **core workflow** and **hands-on practice**. Little to no Cloud setup to get started with Terraform. With just a local computer, time, and commitment, you can build a strong foundation in Terraform.

This guide is my way of **_paying it forward_** to help you pass your exam using free, high-quality resources.


## 📖 How to Use This Guide

- **Navigate by Objective:** Use the links below to dive into specific exam objectives.
- **Deep Dive:** Each section includes ✅ key concepts, 🛠️ HCL snippets, 📌 Exam tips, and 🔗 direct links to official documentation.
 - **Test Yourself:** At the end of every objective, there is a **Mini Quiz** and an **Up to 20-Question Quiz**. Following the [Terraform exam sample format](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004), use the `Show Answers` toggles to check your responses.
- **Review Regularly:** Revisit sections as needed to reinforce your understanding and retention.
- **Pace Yourself:** Set a study schedule that allows you to cover all topics without cramming.
- **Have Fun:** Enjoy the learning process and celebrate your progress along the way!


## 📚 Table of Contents

1. [Understand Infrastructure as Code (IaC) Concepts](./01-Infrastructure-as-Code-with-Terraform.md)
2. [Understand Terraform’s Purpose](./02-Terraform-fundamentals.md)
3. [Understand Terraform Basics](./03-Core-Terraform-workflow.md)
4. [Use Terraform CLI: Part-1](./04-Terraform-configuration-part-1.md)
5. [Use Terraform CLI: Part-2](./04-Terraform-configuration-part-2.md)
6. [Terraform Modules](./05-Terraform-modules.md)
7. [Terraform State Management](./06-Terraform-state-management.md)
8. [Maintain Infrastructure with Terraform](./07-Maintain-infrastructure-with-Terraform.md)
9. [Terraform Cloud (HCP Terraform)](./08-HCP-Terraform-1.md)


## 🛠️ Example Snippets

```hcl
# Basic resource example
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}
```

When you feel ready, take the **Up to 20-Question Quiz** for each objective to test your knowledge.
## 🔗 Official Exam Resources

- [Exam Objectives (004)](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-review-004)
- [Official Study Guide](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004)
- [Sample Questions](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004)


## 📄 License

MIT License — free to use, share, and adapt.


## 🤝 Contributions

Pull requests welcome! If you spot errors or want to add examples, please contribute.

## 🗫 Discussions

Join the conversation on GitHub Discussions to ask questions, share insights, and connect with other learners. What concerns do you have about the exam? Which topics would you like covered in more depth? Let's collaborate and support each other on this learning journey.