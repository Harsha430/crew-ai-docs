# System Design: Enterprise Architectures

**Design production-ready CrewAI systems**

---

## System 1: Enterprise Document Analysis

**Requirements**:
- Process 10,000 documents/day
- Extract structured data + analysis
- Quality validation
- Human approval for high-risk items
- Audit trail for compliance

### Architecture

```
Document → Queue → Extraction Crew → Validation → Approval → Database
             ↓
         (parallel processing)
             ↓
      Analysis Crew → Storage
```

### Implementation

```python
class DocumentAnalysisFlow(Flow):
    @start()
    def start(self):
        docs = load_documents_from_queue()
        return {"documents": docs, "processed": 0}
    
    @listen(start)
    def extract_batch(self, state):
        """Process documents in parallel"""
        extraction_crew = Crew(
            agents=[extractor_agent],
            tasks=[extraction_task]
        )
        
        results = []
        for doc in state["documents"]:
            result = extraction_crew.kickoff(inputs={"document": doc})
            results.append(result)
        
        self.state["extractions"] = results
        return results
    
    @router(extract_batch)
    def route_by_risk(self, results):
        """Route based on risk level"""
        high_risk = [r for r in results if r.confidence < 0.7]
        low_risk = [r for r in results if r.confidence >= 0.7]
        
        self.state["high_risk_count"] = len(high_risk)
        self.state["low_risk_count"] = len(low_risk)
        
        return "human_approval" if high_risk else "auto_approve"
    
    @listen("human_approval")
    def human_review(self):
        """Queue for human review"""
        results = queue_for_human_review(self.state["extractions"])
        self.state["human_approved"] = results
        return "analyze"
    
    @listen("auto_approve")
    def auto_approve(self):
        """Auto-approve low-risk items"""
        self.state["human_approved"] = self.state["extractions"]
        return "analyze"
    
    @listen("analyze")
    def analyze_approved(self):
        """Analyze approved extractions"""
        analysis_crew = Crew(
            agents=[analysis_agent],
            tasks=[analysis_task]
        )
        
        results = []
        for extraction in self.state["human_approved"]:
            result = analysis_crew.kickoff(inputs=extraction)
            results.append(result)
        
        return "store"
    
    @listen("store")
    def store_results(self):
        """Store in database"""
        for result in self.state.get("human_approved", []):
            database.store(result)
        
        log_audit_trail(self.state)
        return {"status": "complete", "processed": len(self.state["human_approved"])}

# Flow statistics
flow = DocumentAnalysisFlow()
result = flow.kickoff()
# Expected: 10,000 docs/day = ~7 docs/minute
```

### Monitoring

```python
# Monitor
- Documents processed: 10,000/day
- Success rate: >99%
- Average latency: <6 seconds per document
- Cost: <$0.05 per document
- Human approval rate: <5% (high-risk)
- Audit trail: 100% logged
```

---

## System 2: Customer Support Escalation

**Requirements**:
- 24/7 support availability
- Tier 1: FAQ/common issues (AI)
- Tier 2: Complex problems (AI + tools)
- Tier 3: Escalation to human agent
- Satisfaction tracking

### Architecture

```
Customer Query
    ↓
Tier 1 Crew (FAQ, common)
    ├─ Can resolve? → Close ticket
    └─ Cannot resolve → Tier 2
    ↓
Tier 2 Crew (Complex, tools)
    ├─ Can resolve? → Close ticket
    └─ Cannot resolve → Tier 3
    ↓
Tier 3 (Human Agent)
    └─ Manual resolution
    ↓
Feedback & Satisfaction
```

### Implementation

```python
class SupportFlow(Flow):
    @start()
    def start(self):
        ticket = get_next_support_ticket()
        self.state["ticket"] = ticket
        self.state["tier"] = 1
        self.state["history"] = []
        return ticket
    
    @listen(start)
    def tier1_crew(self, ticket):
        """Try common issues"""
        tier1 = Crew(
            agents=[faq_agent, common_issues_agent],
            tasks=[answer_task]
        )
        
        result = tier1.kickoff(inputs={"question": ticket["content"]})
        self.state["history"].append(("tier1", result))
        
        return result
    
    @router(tier1_crew)
    def can_tier1_resolve(self, result):
        """Check if tier 1 resolved"""
        if "resolved" in result.raw or result.pydantic.confidence > 0.9:
            return "close"
        else:
            self.state["tier"] = 2
            return "tier2"
    
    @listen("tier2")
    def tier2_crew(self, ticket):
        """Try complex resolution"""
        tier2 = Crew(
            agents=[specialist_agent, tool_agent],
            tasks=[analysis_task, resolution_task],
            process=Process.sequential
        )
        
        result = tier2.kickoff(inputs={
            "question": ticket["content"],
            "tier1_attempt": self.state["history"][0][1]
        })
        self.state["history"].append(("tier2", result))
        
        return result
    
    @router(tier2_crew)
    def can_tier2_resolve(self, result):
        """Check if tier 2 resolved"""
        if "resolved" in result.raw or result.pydantic.confidence > 0.8:
            return "close"
        else:
            self.state["tier"] = 3
            return "escalate"
    
    @listen("escalate")
    def escalate_to_human(self):
        """Queue for human agent"""
        ticket_id = self.state["ticket"]["id"]
        agent_history = "\n".join([
            f"Tier {t[0]}: {t[1].raw[:200]}..."
            for t in self.state["history"]
        ])
        
        escalate_ticket(ticket_id, summary=agent_history)
        return "wait_human"
    
    @listen("wait_human")
    def wait_for_human_resolution(self):
        """Wait for human resolution"""
        return "closed"
    
    @listen("close")
    def close_ticket(self):
        """Close and get feedback"""
        ticket_id = self.state["ticket"]["id"]
        satisfaction = get_customer_feedback(ticket_id)
        
        log_ticket_resolution({
            "ticket_id": ticket_id,
            "tier": self.state["tier"],
            "satisfaction": satisfaction
        })
        
        return {"status": "closed", "satisfaction": satisfaction}

# Flow execution
flow = SupportFlow()
result = flow.kickoff()
```

### Metrics

```python
# Tier 1 Resolution Rate: 70% (common issues)
# Tier 2 Resolution Rate: 20% (complex issues)
# Escalation Rate: 10% (human agents)
# Avg Resolution Time: 2 min (tier 1) / 5 min (tier 2)
# Customer Satisfaction: 4.5/5.0
# Cost per Ticket: $0.50 (AI) / $15 (human)
```

---

## System 3: Multi-Source Data Aggregation

**Requirements**:
- Aggregate data from multiple sources
- Transform to common format
- Detect conflicts/duplicates
- Combine into single view
- Real-time updates

### Architecture

```
Source 1 (API)
Source 2 (Database)      → Transform Crew → Combine Crew → Conflict Resolution → Output
Source 3 (File)
Source 4 (Stream)
```

### Implementation

```python
class DataAggregationCrew:
    def __init__(self):
        self.sources = {
            "api": APISource(),
            "db": DatabaseSource(),
            "file": FileSource(),
            "stream": StreamSource()
        }
        
        # Crews for each stage
        self.transform_crew = Crew(
            agents=[transform_agent],
            tasks=[transform_task]
        )
        
        self.combine_crew = Crew(
            agents=[combine_agent, conflict_resolver],
            tasks=[combine_task, resolve_conflicts_task]
        )
    
    def aggregate(self) -> dict:
        """Aggregate from all sources"""
        
        # Fetch from all sources
        data = {}
        for source_name, source in self.sources.items():
            try:
                data[source_name] = source.fetch()
            except Exception as e:
                log_error(f"Failed to fetch from {source_name}: {e}")
                data[source_name] = None
        
        # Transform each source
        transformed = {}
        for source_name, source_data in data.items():
            if source_data:
                result = self.transform_crew.kickoff(inputs={
                    "source": source_name,
                    "data": source_data
                })
                transformed[source_name] = result
        
        # Combine and resolve conflicts
        combined = self.combine_crew.kickoff(inputs={
            "sources": transformed
        })
        
        return combined

# Usage
aggregator = DataAggregationCrew()
combined_data = aggregator.aggregate()
```

---

## System 4: Autonomous Research & Publishing

**Requirements**:
- Research trending topics
- Synthesize findings
- Generate article
- Quality review
- Auto-publish
- Track engagement

### Architecture

```
Topic Trends
    ↓
Research Team (3 agents)
    ├─ Primary research
    ├─ Secondary validation
    └─ Source verification
    ↓
Analysis Team (2 agents)
    ├─ Synthesize findings
    └─ Identify key insights
    ↓
Writing Team (2 agents)
    ├─ Draft article
    └─ Edit & format
    ↓
Review (Human + AI)
    ├─ Fact check
    ├─ Tone review
    └─ Compliance
    ↓
Publishing
    └─ Auto-publish if all pass
    ↓
Monitoring
    ├─ Engagement metrics
    └─ Feedback
```

### Implementation

```python
class ResearchPublishingFlow(Flow):
    @start()
    def start(self):
        trending_topics = get_trending_topics()
        topic = select_best_topic(trending_topics)
        
        self.state["topic"] = topic
        self.state["stage"] = "research"
        self.state["quality_scores"] = {}
        
        return topic
    
    @listen(start)
    def research_team(self, topic):
        """Multi-agent research"""
        research_crew = Crew(
            agents=[primary_researcher, secondary_researcher, source_validator],
            tasks=[research_task1, research_task2, validation_task],
            process=Process.hierarchical,
            memory=True
        )
        
        result = research_crew.kickoff(inputs={"topic": topic})
        self.state["research"] = result
        self.state["stage"] = "analysis"
        
        return result
    
    @listen(research_team)
    def analysis_team(self, research_result):
        """Synthesize & analyze"""
        analysis_crew = Crew(
            agents=[synthesizer, insight_extractor],
            tasks=[synthesis_task, insight_task],
            context=[research_result]  # ← Use research as context
        )
        
        result = analysis_crew.kickoff()
        self.state["analysis"] = result
        self.state["stage"] = "writing"
        
        return result
    
    @listen(analysis_team)
    def writing_team(self, analysis_result):
        """Draft article"""
        writing_crew = Crew(
            agents=[writer, editor],
            tasks=[write_task, edit_task],
            context=[analysis_result]
        )
        
        result = writing_crew.kickoff()
        self.state["article"] = result
        self.state["stage"] = "review"
        
        return result
    
    @router(writing_team)
    def route_to_review(self, article):
        """Send for review"""
        # Could route to human or AI review
        return "auto_review"
    
    @listen("auto_review")
    def auto_review(self):
        """AI review"""
        review_crew = Crew(
            agents=[fact_checker, tone_reviewer],
            tasks=[fact_check_task, tone_check_task],
            context=[self.state["article"]]
        )
        
        result = review_crew.kickoff()
        self.state["review"] = result
        
        # Score quality
        self.state["quality_scores"]["facts"] = score_facts(result)
        self.state["quality_scores"]["tone"] = score_tone(result)
        
        avg_score = sum(self.state["quality_scores"].values()) / len(self.state["quality_scores"])
        
        if avg_score > 0.8:
            return "publish"
        else:
            return "revise"
    
    @listen("revise")
    def revise_article(self):
        """Send back to writing team"""
        # In production: queue for manual revision or re-run writing crew
        self.state["stage"] = "writing"
        return "finished"
    
    @listen("publish")
    def publish_article(self):
        """Auto-publish"""
        article = self.state["article"]
        
        article_id = publish_to_platform(article.raw)
        
        self.state["article_id"] = article_id
        self.state["published_at"] = datetime.now()
        self.state["stage"] = "monitoring"
        
        return "finished"
    
    @listen("finished")
    def finish(self):
        """Complete"""
        return {
            "topic": self.state["topic"],
            "article_id": self.state.get("article_id"),
            "quality_score": sum(self.state["quality_scores"].values()) / len(self.state["quality_scores"]),
            "published": self.state["stage"] == "monitoring"
        }

# Flow execution
flow = ResearchPublishingFlow()
result = flow.kickoff()

# Monitoring
monitor_article_engagement(result["article_id"])
```

---

## System Design Principles

✅ **DO:**
- Clear separation of concerns
- Reusable crews/flows
- Error handling at each stage
- Monitoring & observability
- Graceful degradation
- Cost optimization
- Scale for peak load
- Human-in-loop where needed

❌ **DON'T:**
- Monolithic everything in one crew
- Hardcoded values (use config)
- Ignore failure scenarios
- Skip logging/monitoring
- Assume things always work
- Overprovision resources
- Mix business logic with orchestration
- Skip testing in staging

---

## Complete Example: Sales AI Assistant

```python
# Full end-to-end system

class SalesAssistantFlow(Flow):
    @start()
    def start(self):
        # Get customer info from CRM
        customer = get_customer_from_crm()
        return {"customer": customer}
    
    @listen(start)
    def analyze_customer(self, data):
        # Understand customer needs
        crew = Crew(agents=[customer_analyst], tasks=[analysis_task])
        result = crew.kickoff(inputs=data)
        self.state["analysis"] = result
        return result
    
    @router(analyze_customer)
    def route_by_segment(self, analysis):
        # Route based on customer segment
        segment = analysis.pydantic.segment  # "enterprise", "smb", "startup"
        return segment
    
    @listen("enterprise")
    def enterprise_sales(self):
        # Enterprise sales process
        crew = Crew(
            agents=[enterprise_sales_agent],
            tasks=[proposal_task],
            context=[self.state["analysis"]]
        )
        return crew.kickoff()
    
    @listen("smb")
    def smb_sales(self):
        # SMB sales process
        crew = Crew(
            agents=[smb_sales_agent],
            tasks=[demo_task],
            context=[self.state["analysis"]]
        )
        return crew.kickoff()
    
    @listen("startup")
    def startup_sales(self):
        # Startup sales process
        crew = Crew(
            agents=[startup_growth_agent],
            tasks=[trial_setup_task],
            context=[self.state["analysis"]]
        )
        return crew.kickoff()

# Execute
flow = SalesAssistantFlow()
result = flow.kickoff()
```

---

## Summary

System design is about:
1. **Clear structure** - Separate concerns into flows/crews/tasks
2. **Scalability** - Handle growth in volume/complexity
3. **Reliability** - Fail gracefully, monitor everything
4. **Cost** - Optimize for efficiency
5. **Maintainability** - Code future engineers can understand
6. **Observability** - See what's happening
7. **Human Control** - Keep humans in the loop where needed

---

**End of Documentation**
