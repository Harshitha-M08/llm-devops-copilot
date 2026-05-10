# Fixing Option 1: Action Mapping Logic

## Context: What's NOT Happening

### Current Problem
The **Analyzer Agent** is successfully performing LLM-based root cause analysis of incidents with 70% confidence, BUT it's generating **0 actionable recommendations**. This breaks the auto-healing chain.

### Evidence from Logs
```
analyzer-agent  | 2025-10-30 09:11:53,634 - __main__ - WARNING - Could not map action to type: Investigate the test-app service for potential memory leaks
analyzer-agent  | 2025-10-30 09:11:53,634 - __main__ - WARNING - Could not map action to type: Check the user traffic to the test-app service
analyzer-agent  | 2025-10-30 09:11:53,634 - __main__ - WARNING - Could not map action to type: Review the code of the test-app service for potential bugs causing excessive memory consumption
analyzer-agent  | 2025-10-30 09:11:53,634 - __main__ - INFO - ✓ Generated 0 recommendations

auto-response-agent  | 2025-10-30 09:11:53,641 - __main__ - INFO - Processing analysis for incident 4e0a16af-f3b4-41b8-8599-0073067f617e: 0 recommendations
auto-response-agent  | 2025-10-30 09:11:53,641 - __main__ - INFO - No recommendations for incident 4e0a16af-f3b4-41b8-8599-0073067f617e, skipping
```

### Root Cause
The LLM (GPT-4/Claude) is generating **natural language suggestions** like:
- "Investigate the test-app service for potential memory leaks"
- "Check the user traffic to the test-app service"
- "Review the code of the test-app service"

But the **action mapper function** in the analyzer-agent is:
1. Unable to parse these natural language suggestions
2. Not converting them to structured action types (scale/restart/rollback)
3. Returning 0 recommendations to auto-response-agent

### Impact
- ✅ Incident Detection: Working
- ✅ LLM Analysis: Working (70% confidence)
- ❌ **Action Generation: BROKEN (0 recommendations)**
- ❌ Auto-Healing: Not executing (no actions to execute)

---

## Investigation Plan

### Phase 1: Locate the Action Mapping Code (15 minutes)

#### Step 1.1: Find the Analyzer Agent's Action Mapper
**File to examine:** `services/analyzer-agent/app/main.py`

**What to look for:**
- Function that processes LLM output and extracts recommendations
- Code that maps natural language → action types
- Logic that validates action types (scale, restart, rollback, etc.)
- The function that logs "Could not map action to type"

**Commands to run:**
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
grep -r "Could not map action to type" . -n
grep -r "map.*action.*type" . -n -i
grep -r "recommendation" app/ -n -i
```

#### Step 1.2: Examine LLM Integration Code
**Files to examine:**
- `services/analyzer-agent/app/llm_analyzer.py` or similar
- `services/analyzer-agent/app/prompts.py` (if exists)

**What to look for:**
- The prompt sent to the LLM
- Expected LLM response format (JSON? Text? Structured?)
- How recommendations are extracted from LLM response

**Commands to run:**
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
find . -name "*llm*.py" -o -name "*prompt*.py"
grep -r "recommendations" app/ -A 5 -B 5
```

#### Step 1.3: Check Data Models
**Files to examine:**
- `services/analyzer-agent/app/models.py` or `schemas.py`

**What to look for:**
- `Recommendation` data model structure
- `ActionType` enum or constants
- Required fields for a valid recommendation

**Commands to run:**
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
grep -r "class Recommendation" . -n
grep -r "ActionType" . -n
grep -r "restart\|scale\|rollback" . -n
```

---

### Phase 2: Understand Current Implementation (30 minutes)

#### Step 2.1: Analyze the Action Mapping Function
**Questions to answer:**
1. What is the exact function signature?
2. What input does it receive? (LLM raw text? Parsed JSON?)
3. What output should it produce? (List of Recommendation objects?)
4. What are the valid action types? (Enum values? String constants?)
5. What mapping logic is used? (Regex? Keyword matching? ML model?)

**Create analysis document:**
```markdown
## Current Action Mapping Implementation

### Function Location
File: `services/analyzer-agent/app/<filename>.py`
Line: <line_number>

### Function Signature
```python
def map_action_to_type(action_text: str) -> Optional[ActionType]:
    ...
```

### Valid Action Types
- SCALE
- RESTART
- ROLLBACK
- (others?)

### Current Mapping Logic
[Describe how it works]

### Why It's Failing
[Root cause analysis]
```

#### Step 2.2: Review the LLM Prompt
**Questions to answer:**
1. Does the prompt explicitly ask for structured recommendations?
2. Does it provide examples of valid action types?
3. Does it request JSON format or free-form text?
4. Is the prompt optimized for GPT-4/Claude output parsing?

**Example of what we might find:**
```python
# BAD: Unstructured prompt
prompt = f"Analyze this incident: {incident_data}. What should we do?"

# GOOD: Structured prompt
prompt = f"""Analyze this incident and provide recommendations in JSON format:
{{
  "recommendations": [
    {{
      "action_type": "RESTART|SCALE|ROLLBACK",
      "target_service": "service-name",
      "parameters": {{}},
      "rationale": "why this action"
    }}
  ]
}}

Incident: {incident_data}
"""
```

#### Step 2.3: Test the Current Mapper Manually
**Create a test script:** `test_action_mapper.py`

```python
# Test the current action mapper with sample LLM outputs
from app.main import map_action_to_type  # or wherever it lives

test_cases = [
    "Restart the test-app service",
    "Scale up the pods to 3 replicas",
    "Rollback to previous deployment",
    "Investigate the test-app service for potential memory leaks",  # This is failing
    "Check the user traffic to the test-app service",  # This is failing
]

for test_input in test_cases:
    result = map_action_to_type(test_input)
    print(f"Input: {test_input}")
    print(f"Output: {result}")
    print("---")
```

**Run the test:**
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
python test_action_mapper.py
```

---

### Phase 3: Design the Fix (45 minutes)

#### Step 3.1: Decide on the Approach

**Option A: Improve the LLM Prompt (Recommended)**
- Modify the prompt to explicitly request structured JSON output
- Provide examples of valid recommendations in the prompt
- Use few-shot learning with example incident → recommendation pairs
- Add constraints: "Only suggest RESTART, SCALE, or ROLLBACK actions"

**Pros:**
- Leverages LLM's capability better
- More reliable structured output
- Easier to maintain

**Cons:**
- Still depends on LLM following instructions
- May need prompt engineering iteration

**Option B: Improve the Action Mapper (Fallback)**
- Keep current prompt
- Build a smarter regex/NLP-based mapper
- Use keyword matching: "restart" → RESTART, "scale" → SCALE
- Extract service names from natural language

**Pros:**
- Works with any LLM output format
- More forgiving of LLM variations

**Cons:**
- Complex regex logic
- Prone to false positives/negatives
- Harder to maintain

**Option C: Hybrid Approach (Best)**
1. Improve LLM prompt for structured output
2. Add fallback action mapper for natural language
3. Validate mapped actions before returning

**Recommendation: Use Option C**

#### Step 3.2: Design the New LLM Prompt

**File to modify:** `services/analyzer-agent/app/prompts.py` (or wherever prompts are defined)

**New Prompt Structure:**
```python
ANALYSIS_PROMPT = """
You are a DevOps AI agent analyzing infrastructure incidents.

INCIDENT DATA:
- Service: {service_name}
- Metric: {metric_name}
- Current Value: {current_value}
- Severity: {severity}
- Recent Logs: {logs}
- Historical Context: {rag_context}

YOUR TASK:
1. Identify the root cause
2. Assess confidence level (0-100%)
3. Provide ACTIONABLE recommendations using ONLY these action types:
   - RESTART: Restart pods/containers
   - SCALE: Scale up/down replicas
   - ROLLBACK: Rollback to previous version
   - INVESTIGATE: Requires human investigation (no auto-healing)

RESPONSE FORMAT (strict JSON):
{{
  "root_cause": "Brief description of root cause",
  "confidence": 85,
  "recommendations": [
    {{
      "action_type": "RESTART",
      "target_service": "test-app",
      "target_type": "deployment",
      "parameters": {{}},
      "rationale": "High memory usage indicates memory leak, restart will clear it",
      "criticality": "medium",
      "estimated_impact": "30 seconds downtime"
    }}
  ]
}}

EXAMPLES:

Example 1 - High CPU:
{{
  "root_cause": "CPU spike due to infinite loop in processing",
  "confidence": 75,
  "recommendations": [
    {{
      "action_type": "RESTART",
      "target_service": "api-service",
      "target_type": "deployment",
      "parameters": {{}},
      "rationale": "Restart will kill stuck processes",
      "criticality": "high",
      "estimated_impact": "Brief service interruption"
    }}
  ]
}}

Example 2 - Memory Leak:
{{
  "root_cause": "Memory leak causing OOM errors",
  "confidence": 80,
  "recommendations": [
    {{
      "action_type": "RESTART",
      "target_service": "worker-service",
      "target_type": "deployment",
      "parameters": {{}},
      "rationale": "Clear leaked memory",
      "criticality": "medium",
      "estimated_impact": "30 seconds restart time"
    }},
    {{
      "action_type": "SCALE",
      "target_service": "worker-service",
      "target_type": "deployment",
      "parameters": {{"replicas": 3}},
      "rationale": "Distribute load while investigating",
      "criticality": "low",
      "estimated_impact": "Increased resource usage"
    }}
  ]
}}

IMPORTANT RULES:
- Only use action_type values: RESTART, SCALE, ROLLBACK, INVESTIGATE
- Always include target_service and target_type
- Criticality must be: low, medium, high, critical
- If uncertain what action to take, use action_type: INVESTIGATE
- Return valid JSON only, no markdown code blocks
- If no safe automated action exists, return empty recommendations array

Now analyze the incident above.
"""
```

#### Step 3.3: Design the Enhanced Action Mapper

**File to create/modify:** `services/analyzer-agent/app/action_mapper.py`

```python
from enum import Enum
from typing import List, Optional, Dict, Any
import re
import json
from pydantic import BaseModel, ValidationError

class ActionType(str, Enum):
    RESTART = "restart"
    SCALE = "scale"
    ROLLBACK = "rollback"
    INVESTIGATE = "investigate"

class Recommendation(BaseModel):
    action_type: ActionType
    target_service: str
    target_type: str = "deployment"
    parameters: Dict[str, Any] = {}
    rationale: str
    criticality: str = "medium"
    estimated_impact: str = ""

class ActionMapper:
    """Maps LLM output to structured recommendations"""

    @staticmethod
    def parse_llm_response(llm_output: str) -> List[Recommendation]:
        """
        Parse LLM output and extract recommendations.
        Tries multiple parsing strategies:
        1. JSON parsing (preferred)
        2. Regex extraction (fallback)
        3. Keyword matching (last resort)
        """
        recommendations = []

        # Strategy 1: Try JSON parsing
        try:
            recommendations = ActionMapper._parse_json(llm_output)
            if recommendations:
                return recommendations
        except Exception as e:
            print(f"JSON parsing failed: {e}")

        # Strategy 2: Try regex extraction
        try:
            recommendations = ActionMapper._parse_with_regex(llm_output)
            if recommendations:
                return recommendations
        except Exception as e:
            print(f"Regex parsing failed: {e}")

        # Strategy 3: Keyword matching
        try:
            recommendations = ActionMapper._parse_with_keywords(llm_output)
            if recommendations:
                return recommendations
        except Exception as e:
            print(f"Keyword parsing failed: {e}")

        return []

    @staticmethod
    def _parse_json(llm_output: str) -> List[Recommendation]:
        """Parse structured JSON response from LLM"""
        # Remove markdown code blocks if present
        cleaned = re.sub(r'```json\n|\n```|```', '', llm_output)

        # Parse JSON
        data = json.loads(cleaned)

        # Extract recommendations
        recommendations = []
        if 'recommendations' in data:
            for rec_data in data['recommendations']:
                try:
                    rec = Recommendation(**rec_data)
                    recommendations.append(rec)
                except ValidationError as e:
                    print(f"Invalid recommendation format: {e}")

        return recommendations

    @staticmethod
    def _parse_with_regex(llm_output: str) -> List[Recommendation]:
        """Extract recommendations using regex patterns"""
        recommendations = []

        # Pattern: "RESTART the api-service because..."
        restart_pattern = r'(?:RESTART|restart|Restart)\s+(?:the\s+)?([a-z0-9-]+)(?:\s+service)?.*?(?:because|due to|to)\s+([^.]+)'

        for match in re.finditer(restart_pattern, llm_output, re.IGNORECASE):
            service_name = match.group(1)
            rationale = match.group(2).strip()

            rec = Recommendation(
                action_type=ActionType.RESTART,
                target_service=service_name,
                target_type="deployment",
                parameters={},
                rationale=rationale,
                criticality="medium"
            )
            recommendations.append(rec)

        # Pattern: "SCALE the api-service to 3 replicas"
        scale_pattern = r'(?:SCALE|scale|Scale)\s+(?:the\s+)?([a-z0-9-]+).*?to\s+(\d+)\s+replica'

        for match in re.finditer(scale_pattern, llm_output, re.IGNORECASE):
            service_name = match.group(1)
            replicas = int(match.group(2))

            rec = Recommendation(
                action_type=ActionType.SCALE,
                target_service=service_name,
                target_type="deployment",
                parameters={"replicas": replicas},
                rationale=f"Scale to {replicas} replicas",
                criticality="medium"
            )
            recommendations.append(rec)

        # Add more patterns for ROLLBACK, etc.

        return recommendations

    @staticmethod
    def _parse_with_keywords(llm_output: str) -> List[Recommendation]:
        """Fallback: Extract service names and guess action from keywords"""
        recommendations = []

        # If output contains "restart" and a service name
        if re.search(r'\brestart\b', llm_output, re.IGNORECASE):
            # Extract service name (common patterns: test-app, api-service, etc.)
            service_match = re.search(r'\b([a-z0-9]+-[a-z0-9]+|test-app|api-service|worker-service)\b', llm_output, re.IGNORECASE)

            if service_match:
                service_name = service_match.group(1)
                rec = Recommendation(
                    action_type=ActionType.RESTART,
                    target_service=service_name,
                    target_type="deployment",
                    parameters={},
                    rationale="Extracted from natural language suggestion",
                    criticality="medium"
                )
                recommendations.append(rec)

        return recommendations

    @staticmethod
    def validate_recommendation(rec: Recommendation) -> bool:
        """Validate that recommendation is safe to execute"""
        # Check required fields
        if not rec.target_service:
            return False

        # Validate action type
        if rec.action_type not in ActionType:
            return False

        # Service name must be alphanumeric with hyphens
        if not re.match(r'^[a-z0-9-]+$', rec.target_service):
            return False

        return True
```

---

### Phase 4: Implementation (2 hours)

#### Step 4.1: Backup Current Code
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
cp app/main.py app/main.py.backup
cp app/llm_analyzer.py app/llm_analyzer.py.backup
```

#### Step 4.2: Implement the New Prompt

**File:** `services/analyzer-agent/app/prompts.py` (create if doesn't exist)

1. Create the new prompt template (use design from Phase 3.2)
2. Add dynamic variable substitution
3. Ensure prompt includes examples and constraints

**Implementation checklist:**
- [ ] Create/update prompts.py file
- [ ] Add ANALYSIS_PROMPT constant with new template
- [ ] Add variable placeholders: {service_name}, {metric_name}, etc.
- [ ] Include JSON schema in prompt
- [ ] Add at least 3 examples of valid responses
- [ ] Test prompt renders correctly with sample data

#### Step 4.3: Implement the Enhanced Action Mapper

**File:** `services/analyzer-agent/app/action_mapper.py` (create new file)

1. Copy the ActionMapper class from Phase 3.3
2. Implement all three parsing strategies
3. Add comprehensive error handling
4. Add logging for debugging

**Implementation checklist:**
- [ ] Create action_mapper.py file
- [ ] Implement ActionType enum
- [ ] Implement Recommendation Pydantic model
- [ ] Implement ActionMapper.parse_llm_response()
- [ ] Implement _parse_json() with JSON parsing
- [ ] Implement _parse_with_regex() with regex patterns
- [ ] Implement _parse_with_keywords() as fallback
- [ ] Implement validate_recommendation()
- [ ] Add logging statements
- [ ] Add unit tests

#### Step 4.4: Integrate with Main Analyzer Code

**File:** `services/analyzer-agent/app/main.py`

**Changes needed:**
1. Import new ActionMapper
2. Update LLM call to use new prompt
3. Replace old action mapping logic with ActionMapper
4. Add validation before returning recommendations

**Example integration:**
```python
from app.action_mapper import ActionMapper, Recommendation
from app.prompts import ANALYSIS_PROMPT

async def analyze_incident(incident_data):
    # ... existing code ...

    # Format prompt with incident data
    prompt = ANALYSIS_PROMPT.format(
        service_name=incident_data['service'],
        metric_name=incident_data['metric'],
        current_value=incident_data['value'],
        severity=incident_data['severity'],
        logs=recent_logs,
        rag_context=similar_incidents
    )

    # Call LLM
    llm_response = await llm_client.generate(prompt)

    # Parse recommendations using new mapper
    recommendations = ActionMapper.parse_llm_response(llm_response)

    # Validate recommendations
    valid_recommendations = [
        rec for rec in recommendations
        if ActionMapper.validate_recommendation(rec)
    ]

    logger.info(f"✓ Generated {len(valid_recommendations)} valid recommendations")

    # ... publish to RabbitMQ ...
```

**Implementation checklist:**
- [ ] Import ActionMapper and new prompts
- [ ] Replace prompt string with new ANALYSIS_PROMPT
- [ ] Replace old mapping logic with ActionMapper.parse_llm_response()
- [ ] Add validation loop
- [ ] Update logging statements
- [ ] Handle empty recommendations gracefully
- [ ] Test integration

#### Step 4.5: Update Environment Variables (if needed)

**File:** `.env` or `services/analyzer-agent/.env`

Check if any new configuration is needed:
- LLM model selection (GPT-4 vs Claude)
- Temperature settings
- Max tokens
- Timeout values

#### Step 4.6: Update Requirements

**File:** `services/analyzer-agent/requirements.txt`

Ensure all dependencies are listed:
```
pydantic>=2.0.0
```

---

### Phase 5: Testing (1.5 hours)

#### Step 5.1: Unit Tests

**File:** `services/analyzer-agent/tests/test_action_mapper.py`

```python
import pytest
from app.action_mapper import ActionMapper, ActionType, Recommendation

def test_parse_json_valid():
    """Test parsing valid JSON response"""
    llm_output = '''
    {
      "root_cause": "Memory leak",
      "confidence": 80,
      "recommendations": [
        {
          "action_type": "RESTART",
          "target_service": "test-app",
          "target_type": "deployment",
          "parameters": {},
          "rationale": "Clear leaked memory",
          "criticality": "medium"
        }
      ]
    }
    '''

    recommendations = ActionMapper.parse_llm_response(llm_output)

    assert len(recommendations) == 1
    assert recommendations[0].action_type == ActionType.RESTART
    assert recommendations[0].target_service == "test-app"

def test_parse_json_with_markdown():
    """Test parsing JSON wrapped in markdown code blocks"""
    llm_output = '''
    Here's my analysis:

    ```json
    {
      "recommendations": [
        {
          "action_type": "SCALE",
          "target_service": "api-service",
          "target_type": "deployment",
          "parameters": {"replicas": 3},
          "rationale": "Handle increased load",
          "criticality": "low"
        }
      ]
    }
    ```
    '''

    recommendations = ActionMapper.parse_llm_response(llm_output)

    assert len(recommendations) == 1
    assert recommendations[0].action_type == ActionType.SCALE
    assert recommendations[0].parameters["replicas"] == 3

def test_parse_natural_language_restart():
    """Test regex fallback for natural language"""
    llm_output = "I recommend to RESTART the test-app service because it has a memory leak"

    recommendations = ActionMapper.parse_llm_response(llm_output)

    assert len(recommendations) >= 1
    assert recommendations[0].action_type == ActionType.RESTART
    assert recommendations[0].target_service == "test-app"

def test_parse_natural_language_scale():
    """Test regex fallback for scale command"""
    llm_output = "You should SCALE the api-service to 5 replicas to handle the load"

    recommendations = ActionMapper.parse_llm_response(llm_output)

    assert len(recommendations) >= 1
    assert recommendations[0].action_type == ActionType.SCALE
    assert recommendations[0].parameters["replicas"] == 5

def test_parse_invalid_json():
    """Test fallback when JSON is invalid"""
    llm_output = "This is not valid JSON but mentions restart test-app"

    recommendations = ActionMapper.parse_llm_response(llm_output)

    # Should fallback to regex/keyword parsing
    assert isinstance(recommendations, list)

def test_validate_recommendation_valid():
    """Test recommendation validation"""
    rec = Recommendation(
        action_type=ActionType.RESTART,
        target_service="test-app",
        target_type="deployment",
        parameters={},
        rationale="Test",
        criticality="medium"
    )

    assert ActionMapper.validate_recommendation(rec) == True

def test_validate_recommendation_invalid_service():
    """Test validation rejects invalid service names"""
    rec = Recommendation(
        action_type=ActionType.RESTART,
        target_service="invalid service name!",
        target_type="deployment",
        parameters={},
        rationale="Test",
        criticality="medium"
    )

    assert ActionMapper.validate_recommendation(rec) == False
```

**Run unit tests:**
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
pytest tests/test_action_mapper.py -v
```

#### Step 5.2: Integration Test with Real LLM

**File:** `services/analyzer-agent/tests/test_integration.py`

```python
import asyncio
from app.main import analyze_incident
from app.action_mapper import ActionType

async def test_real_incident_analysis():
    """Test with real incident data"""
    incident_data = {
        'incident_id': 'test-123',
        'service': 'test-app',
        'metric': 'memory_mb',
        'value': 90.5,
        'severity': 'high',
        'timestamp': '2025-10-30T09:00:00Z'
    }

    result = await analyze_incident(incident_data)

    print(f"\nRoot Cause: {result['root_cause']}")
    print(f"Confidence: {result['confidence']}%")
    print(f"Recommendations: {len(result['recommendations'])}")

    for rec in result['recommendations']:
        print(f"  - {rec.action_type}: {rec.target_service} - {rec.rationale}")

    # Assertions
    assert result['confidence'] > 0
    assert len(result['recommendations']) > 0
    assert all(rec.action_type in ActionType for rec in result['recommendations'])

if __name__ == "__main__":
    asyncio.run(test_real_incident_analysis())
```

**Run integration test:**
```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
python -m tests.test_integration
```

#### Step 5.3: End-to-End Test in Docker

**Rebuild and restart analyzer-agent:**
```bash
cd LLM DevOps Copilot-main
docker-compose build analyzer-agent
docker-compose restart analyzer-agent
```

**Trigger a new incident:**
```bash
curl -X POST http://localhost:8080/trigger-memory
```

**Monitor the logs:**
```bash
# Watch analyzer logs
docker-compose logs -f analyzer-agent | grep -E "recommendation|WARNING|ERROR"

# After 30 seconds, check auto-response logs
docker-compose logs -f auto-response-agent | grep -E "recommendation|Processing|Executing"
```

**Expected successful output:**
```
analyzer-agent: ✓ Generated 2 valid recommendations
analyzer-agent: 📊 Analysis published: Memory leak detected | Confidence: 85% | Recommendations: 2
auto-response-agent: Processing analysis for incident xxx: 2 recommendations
auto-response-agent: ✓ Recommendation 1: RESTART test-app (criticality: medium)
```

#### Step 5.4: Verify in Database

```bash
# Check that recommendations are stored
cd LLM DevOps Copilot-main
docker-compose exec -T postgres psql -U postgres -d devopsdb -c "SELECT COUNT(*) FROM recommendations;"
```

---

### Phase 6: Validation & Rollout (30 minutes)

#### Step 6.1: Success Criteria Checklist

- [ ] Analyzer-agent no longer logs "Could not map action to type" warnings
- [ ] `recommendation_count` in logs shows > 0 (e.g., "Generated 2 recommendations")
- [ ] Auto-response-agent logs show "Processing analysis: X recommendations" where X > 0
- [ ] LLM consistently returns structured JSON responses
- [ ] Regex fallback works when JSON parsing fails
- [ ] At least 80% of incidents generate 1+ recommendations
- [ ] No crashes or exceptions in analyzer-agent
- [ ] Recommendations pass validation (valid action types, service names)

#### Step 6.2: Monitor Production Metrics

**Grafana Dashboard to check:**
- `analyzer_recommendations_generated_total` metric
- `analyzer_mapping_failures_total` metric (should decrease)
- `autoresponse_actions_received_total` metric (should increase)

**Access Grafana:**
```bash
# Open browser to http://localhost:3002
# Default credentials: admin/admin
# Look for "Analyzer Agent" dashboard
```

#### Step 6.3: Rollback Plan (if needed)

If the fix causes issues:

```bash
cd LLM DevOps Copilot-main/services/analyzer-agent
cp app/main.py.backup app/main.py
cp app/llm_analyzer.py.backup app/llm_analyzer.py
rm app/action_mapper.py
rm app/prompts.py

cd ../..
docker-compose build analyzer-agent
docker-compose restart analyzer-agent
```

---

## Expected Outcome

### Before Fix
```
analyzer-agent: WARNING - Could not map action to type: Investigate...
analyzer-agent: INFO - ✓ Generated 0 recommendations
auto-response-agent: INFO - No recommendations for incident, skipping
```

### After Fix
```
analyzer-agent: INFO - ✓ Parsed LLM response: 2 recommendations found
analyzer-agent: INFO - ✓ Validated recommendation: RESTART test-app
analyzer-agent: INFO - ✓ Validated recommendation: SCALE test-app to 3 replicas
analyzer-agent: INFO - ✓ Generated 2 valid recommendations
analyzer-agent: INFO - 📊 Analysis published: Memory leak | Confidence: 85% | Recommendations: 2

auto-response-agent: INFO - Processing analysis: 2 recommendations
auto-response-agent: INFO - Evaluating recommendation: RESTART test-app (criticality: medium)
auto-response-agent: INFO - ✓ Decision: Execute automatically (confidence: 85%)
```

---

## Timeline Estimate

- **Phase 1 (Investigation):** 15 minutes
- **Phase 2 (Understanding):** 30 minutes
- **Phase 3 (Design):** 45 minutes
- **Phase 4 (Implementation):** 2 hours
- **Phase 5 (Testing):** 1.5 hours
- **Phase 6 (Validation):** 30 minutes

**Total: ~5.5 hours**

---

## Dependencies

- LLM service must be running (OpenAI or Anthropic API keys configured)
- RabbitMQ must be operational
- Test-app must be able to generate incidents
- Prometheus must be scraping metrics

---

## Risk Assessment

**Low Risk:**
- Changes are isolated to analyzer-agent
- Fallback parsing strategies reduce failure risk
- Existing functionality preserved (detection, analysis still work)
- Easy rollback with backup files

**Medium Risk:**
- LLM prompt changes might need iteration to get right format
- Different LLM models (GPT-4 vs Claude) may respond differently

**Mitigation:**
- Extensive testing before deployment
- Multiple parsing strategies as fallbacks
- Comprehensive error handling
- Monitoring and alerts

---

## Next Steps After This Fix

Once action mapping is working:
1. Proceed to **Option 2** (Docker Compose executor)
2. Test auto-healing end-to-end
3. Add more action types if needed
4. Tune confidence thresholds
5. Add approval workflows for high-risk actions
