# GNS3 Lab for LFCS: Overview and Guidelines

## Expectation

This GNS3 lab is primarily designed to assess your readiness for the Linux Foundation Certified System Administrator (LFCS) certification. It also serves as a valuable tool for practicing and testing your Linux skills. The lab consists of 20 practical questions that require configuring various aspects of a Linux system and can be completed in approximately 6 hours.

While correctly answering at least 80% of the questions demonstrates a good preparation, it does not guarantee success on the LFCS exam. The exam focuses on evaluating core skills efficiently, within a shorter time frame, emphasizing real-world problem-solving abilities.

This lab offers comprehensive practice, but note that the actual exam may present questions in a more condensed format with less detailed guidance.

### Guidelines

1. There is no single best approach to achieving the desired outcomes; what matters most is answering each question correctly. However, in practice, adhering to best practices and maintaining system security are essential for achieving effective and optimal results.
   
2. You are free to use any suitable applications or packages. For example, in configuring an NTP server, you may use either `ntpd` or `chrony`, as long as it is properly configured.

3. Some tasks are foundational; incomplete configurations may hinder subsequent tasks. For example, if you fail to reset a system password, related tasks cannot proceed.

4. Ensure all configuration changes are persistent. Reboot systems when necessary to confirm functionality is maintained after the restart.

5. During the LFCS exam, using external materials is prohibited. To simulate exam conditions, avoid consulting external resources during this lab.

## Hints

- GNS3 instances are typically accessed via `telnet`, which simulates a terminal environment that defaults to `80` columns and `25` lines. You can adjust the terminal size using:
  ```bash
  stty rows {#rows} columns {#cols}
  # for example
  stty rows 30 columns 180
  ```
  After gaining Internet access, you may install tools like `xterm` or `resize` for automatic adjustment.
  
- Only the **Client Workstation (`client`)** comes with a desktop environment by default. However, once Internet access is available, you can install additional packages, including graphical environments, on other systems as needed.

- Although external resources are restricted in the LFCS exam, Linux systems provide internal documentation. Use `man`, `info`, and the `/usr/share/doc` directory for help and examples.

## Getting Started

- **[Preparation](GNS3 Lab for LFCS - Preparation.md)**: Begin by setting up your environment and reviewing key guidelines.
- **[Questions](GNS3 Lab for LFCS - Questions.md)**: Test your knowledge with real-world practical scenarios.
- **[Questions and Solutions](GNS3 Lab for LFCS - Questions and Solutions.md)**: Compare your answers and deepen your understanding with detailed solutions.
