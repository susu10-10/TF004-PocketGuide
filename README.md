# Terraform Associate 004 Pocket Guide

A concise, structured study material made to assist in preparation for the **HashiCorp Certified: Terraform Associate (004)** exam.  

Reference for this study comes from the **Official Terraform Documentation** and the [**Terraform Associate 004 Learning Path**](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004).


## 💡 What's the Point of This Project?

This guide is something i put together because i understand that the budget for paid courses isn't always available for everyone, or a priority for some. The official docs is without a doubt the best place for learning, but it is easy to get lost when you're a beginner.

I only used the official docs to prepare for my exam, and i passed on the first try. I focused on the **core workflow** and **hands-on practice**, without needing to set up any cloud infrastructure. (although it can help). 

All you require is your own computer, with terraform installed, plus a bit of time and dedication, and you will be able to develop a strong terraform foundation.

In this guide, i am sort of **_Paying it Forward_**, and help you in passing your exam using free and top-quality resources. 

## 📖 How To Get The Most Out Of This Guide?

- **Follow Your Desired Exam Objective:** Go through the links below to get to specific points of the exam.

- **Thorough Explanation:** To make sure you completely grasp the material, every chapter outlines ✅ major topics, 🛠️ HCL codes, 🛠️ Exam tips, and 🔗 links to the official documentation.

- **Test Your Knowledge:** Quite often, the end of each objective is followed by a Mini Quiz and an Up to 20-Question Quiz. In alignment with [Terraform exam sample format](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004), you can use the `Show Answers` to check your answers.

- **Regularly Review:** Go over portions of the text as frequently as you feel necessary to solidify your knowledge and memory.

- **Manage Your Time:** Determine a study timetable that gives you enough time to finish all the subjects without a rush.

- **Enjoy The Process:** Learning is a journey.


## 📚 Table of Contents

1. [Understand Infrastructure as Code (IaC) Concepts](./01-Infrastructure-as-Code-with-Terraform.md)
2. [Understand Terraform’s Purpose](./02-Terraform-fundamentals.md)
3. [Understand Terraform Basics](./03-Core-Terraform-workflow.md)
4. [Use Terraform CLI: Part-1](./04-Terraform-configuration-part-1.md)
5. [Use Terraform CLI: Part-2](./04-Terraform-configuration-part-2.md)
6. [Terraform Modules](./05-Terraform-modules.md)
7. [Terraform State Management](./06-Terraform-state-management.md)
8. [Maintain Infrastructure with Terraform](./07-Maintain-infrastructure-with-terraform.md)
9. [Terraform Cloud (HCP Terraform)](./08-Hcp-Terraform-1.md)
10. [Command Line Appendix](./CLI-Appendix.md)


## 🛠️ Example Snippets

```hcl
# Basic resource example
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}
```

When you have worked on the material enough, take the **Up to 20-Question Quiz** for each objective to challenge and refresh your knowledge. You can find the answers by clicking on the `Show Answers` button at the end of each quiz. 

## 🔗 HashiCorp Resources for Exam

- [Exam Objectives (004)](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-review-004)
- [Official Study Guide](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004)
- [Sample Questions](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004)


## 📄 License

MIT License free to use, share, and modify.


## 🤝 Contributions

Pull requests are very much welcome! Help us by reporting bugs, adding more example, and improving the content. Let's make this guide better together!

