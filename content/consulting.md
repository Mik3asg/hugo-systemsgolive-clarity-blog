+++
title = "Consulting"
description = "Cloud architecture, infrastructure automation, and DevOps consulting"
date = "2026-08-09"
author = "Mickael Asghar"
showdate = false
showreadtime = false
showshare = false
hideSupport = true
comments = false
+++

I help startups and SaaS teams build reliable, scalable, secure and cost-efficient cloud infrastructure through pragmatic architecture, automation and DevOps practices.

I remove complexity by delivering simple and effective solutions.

## Services

<div class="consulting-grid">
  <div class="consulting-card">
    <h3>Cloud Architecture</h3>
    <p>Design and implement resilient, scalable systems on AWS and other cloud platforms.</p>
  </div>
  <div class="consulting-card">
    <h3>Infrastructure as Code</h3>
    <p>Implement Ansible, Terraform, and Packer for reproducible, version-controlled deployments.</p>
  </div>
  <div class="consulting-card">
    <h3>Cost Optimisation on AWS and Azure</h3>
    <p>Right-size compute, storage, and reserved capacity to cut cloud spend without compromising reliability.</p>
  </div>
  <div class="consulting-card">
    <h3>CI/CD &amp; Automation</h3>
    <p>Build deployment pipelines, configuration management, and workflow automation.</p>
  </div>
</div>

<style>
.consulting-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1.25rem;
  margin: 1.5rem 0 2rem;
}
.consulting-card {
  padding: 1.5rem;
  border: 1px solid var(--table-border);
  border-radius: 8px;
  background: var(--choice-bg);
  transition: box-shadow 0.2s var(--ease), transform 0.2s var(--ease);
}
.consulting-card:hover {
  box-shadow: 0 0 2rem var(--shadow);
  transform: translateY(-2px);
}
.consulting-card h3 {
  margin: 0 0 0.5rem;
  color: var(--theme);
  font-size: 1.1rem;
}
.consulting-card p {
  margin: 0;
  color: var(--text);
}
@media (max-width: 42rem) {
  .consulting-grid {
    grid-template-columns: 1fr;
  }
}
</style>
