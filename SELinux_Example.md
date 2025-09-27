# Getting Started with SELinux Policy Management: Real-World Example

This document outlines some foundational steps for 
new Linux users to explore, troubleshoot, and manage 
SELinux policies in a Fedora environment. 
It details a real-world example of resolving a policy-related issue

### 1. Environment Setup

Since SELinux is a Linux-specific technology, our first step
is to create a dedicated, safe environment.

-**Virtual Machine:** A Fedora Server VM was installed using 
virtualization on a macOS host, providing the desired secure, 
dedicated space for experimentation.

-**Essential Tools:** The following tools were installed and enabled
to assist with policy analysis and log collection:
- `setools-console`: a powerful suite of tools for analyzing
SELinux policy, including `seinfo` and `sesearch`.
- `audit`: The audit daemon, `auditd`, was installed and enabled 
to log **all** SELinux events, including denials.

### 2. The Main Challenge: A Denied Action

The goal here was to get a hands-on understanding of how 
SELinux works by triggering a policy denial and then
manipulating the policy to fix it.
- **Hypothesis:** The Apache web server(`httpd`) *should* be denied 
permissions to read a file located in a user's home directory. 
- **Initial Action:** An attempt was made to have the `apathe` user 
read a file in a user's home directory using `sudo -u apache cat ~/testfile.txt`
which held the phrase "This is a test file." within. 
- **Result:** The command was denied as expected! Initially, fixes were 
only made within the SELinux framework to allow this action, but it 
was later found that both the standard Linux permissions also denied 
this action, leading to additional fixes that will be described below. 

### 3. The Troubleshooting Process

The meat of this learning experience was the systematic approach 
to diagnosing the "permission denied" error. 
- **Problem 1: Invisible Denials**
  - **Diagnosis:** Even after checking that `auditd` was installed, 
  the denial was not being seen in the audit logs. This was due to 
  `dontaudit` rules, which are designed to silently ignore 
  certain denials. 
  - **Solution:** utilized `semodule -DB` to temporarily disable 
  `dontaudit` rules, making the denial visible for analysis.
- **Problem 2: Stubborn Denial**
  - **Diagnosis:** Once the command to make the permission was 
  put in place, the attempted `sudo -u apache cat ~/testfile.txt` 
  was still denied. Analysis of standard Linux permissions revealed
  that the `apache` user could not access the home directory by default. 
  - **Solution:** The permissions on teh home directory were changed
  to allow the `apache` user to traverse into it (`chmod o+x ~`). This
  sufficiently resolved the standard Linux issue, allowing SELinux to
  then evaluate the action.

### 4. Resolving the SELinux Policy Denial

Once the policy was clearly logged and able to be studied, a clear 
process was followed to create a policy rule to allow the action. 

**1. Analyze the Denial:** The Access Vector Cache (AVC) denial message 
found in `/var/log/audit/audit.log` explictly stated that the `httpd_t` 
security context was denied `read` and `write` access to a file 
with the `user_home_t` security context. Using the command `audit2allow -w -a` 
also gave the ability to view a concise summary of what was denied, why, 
and how to allow access. 

**2. Generate the Policy Rule:** The `audit2allow` command was 
used to create a policy module from the denial. This command 
automatically translates the denial log into a human-readable
policy rule. 

**3. Inspect the Rule:** The generated file, named `httpd_read_userhome`, 
contained a simple, clear rule:
  >`allow httpd_t user_home_t:file { read getattr };`

**4. Load the Policy:** The compiled policy module
(`httpd_read_userhome.pp`) was then loaded into the system's 
active SELinux policy as a new, active addition with the following 
command: 
> `sudo semodule -i httpd_read_userhome.pp`

**5. Final Test:** Set the mode from permissive to enabled using
`sudo setenforce 1` and verify with the 'sudo sestatus' command,
looking for the output "**Current mode: enforcing**" portion. Then,
run `sudo -u apache cat ~/testfile.txt` once more to now see the text 
"This is a test file" instead of a denial! 



