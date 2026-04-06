Act as a senior Kubernetes expert, DevOps architect, and technical educator.

Explain the topic: "TOPIC" in a highly detailed, structured, and production-grade manner.

Follow these strict requirements:

1. Structure the response as a complete guide with numbered sections (at least 20 sections).
2. Start with a clear introduction explaining what it is and why it is important in Kubernetes architecture.
3. Include a "Core Identity" table with fields like binary name, ports, role, etc.
4. Explain the controller pattern (watch → compare → act → loop).
5. Provide deep explanations of all major built-in controllers:
   - ReplicaSet
   - Deployment
   - Node
   - Service
   - Namespace
   - Job
   - StatefulSet
   - DaemonSet
   - Garbage Collector
   - PersistentVolume
6. Use tables wherever comparison or structured data is needed.
7. Explain internal working concepts like:
   - reconciliation loop
   - informer pattern
   - work queues
8. Include a dedicated section on leader election:
   - why it is needed
   - how it works (Lease object)
   - important flags
   - HA best practices
9. Clearly explain interaction with API server and etcd:
   - emphasize that controllers NEVER talk directly to etcd
10. Add performance tuning section:
   - include important flags
   - concurrency tuning
   - resource limits
11. Add security hardening practices:
   - RBAC
   - TLS
   - service accounts
12. Add monitoring & observability:
   - metrics endpoint
   - important Prometheus metrics
13. Add troubleshooting section with real kubectl commands for:
   - pods not created
   - deployment stuck
   - node NotReady
14. Include disaster recovery concepts:
   - stateless nature of controller manager
   - relation to etcd backups
15. Add interview questions (beginner to advanced) with answers.
16. Add real-world production use cases.
17. Add best practices for production environments.
18. Add common mistakes and pitfalls.
19. Include hands-on labs or mini practical exercises.
20. Provide comparisons with:
   - kube-apiserver
   - kube-scheduler
21. Add a clean ASCII architecture diagram.
22. Include a final cheat sheet section with commands and flags.
23. End with key takeaways and summary.

Formatting requirements:
- Use markdown headings (##, ###)
- Use tables for structured info
- Use bullet points where appropriate
- Use code blocks for commands
- Keep explanations practical, not just theoretical
- Tone should be professional, like documentation used by DevOps engineers

Important:
Do NOT give a short answer. Generate a long, in-depth, production-level explanation similar to official Kubernetes documentation combined with real-world DevOps experience.
