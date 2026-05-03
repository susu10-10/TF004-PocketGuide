# Terraform Associate 004 Pocket Guide

A concise, structured study companion for the **HashiCorp Certified: Terraform Associate (004)** exam.  
This guide is based on the **Official Terraform Documentation** and the [**Terraform Associate 004 Learning Path**](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004) provided by HashiCorp.

---

## 📖 How to Use This Guide

- **Navigate by Objective:** Use the links below to dive into specific exam topics.
- **Deep Dive:** Each section includes ✅ key concepts, 🛠️ HCL snippets, 📌 Exam tips, and 🔗 direct links to official documentation.
- **Test Yourself:** At the end of every objective, there is a **Mini Quiz** and **Up to 20-Question Quiz**. Following [terraform exam sample format](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004), Use the `show answers` toggles to check your answers!
- **Review Regularly:** Revisit sections as needed to reinforce your understanding and retention.
- **Pace Yourself:** Set a study schedule that allows you to cover all topics without cramming.
- **Have Fun:** Enjoy the learning process and celebrate your progress along the way!


## 📚 Table of Contents

1. [Understand Infrastructure as Code (IaC) Concepts](01-Infrastructure-as-Code-with-Terraform.md)
2. [Understand Terraform’s Purpose](02-Terraform-fundamentals.md)
3. [Understand Terraform Basics](03-Core-Terraform-workflow.md)
4. [Use Terraform CLI (outside core workflow)](04-Terraform-configuration.md)
5. [Terraform Modules](05-Terraform-modules.md)
6. [Terraform State Management](06-Terraform-state-management.md)
7. [Maintain Infrastructure with Terraform](07-Maintain-infrastructure-with-Terraform.md)
8. [Terraform Cloud (HCP Terraform)](08-HCP-Terraform.md)

---

## 🛠️ Example Snippets

```hcl
# Basic resource example
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}
```

When you feel ready, you can take the **Up to 20-Question Quiz** for each objective to test your knowledge.
## 🔗 Official Exam Resources

- [Exam Objectives (004)](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-review-004)
- [Official Study Guide](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004)
- [Sample Questions](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004)


## 📄 License

MIT License – free to use, share, and adapt.


## 🤝 Contributions

Pull requests welcome! If you spot errors or want to add examples, feel free to contribute.