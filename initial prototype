import asyncio
import os
import together
from together import AsyncTogether, Together

client = Together(api_key=os.environ.get("TOGETHER_API_KEY"))
async_client = AsyncTogether(api_key=os.environ.get("TOGETHER_API_KEY"))

# Red team specific configuration
user_prompt = """Analyze the following API endpoint for vulnerabilities: 
POST /api/v1/processPayment
Headers: Authorization: Bearer <token>
Body: {'card_number': '4111111111111111', 'expiry': '12/25', 'cvv': '123'}"""

reference_models = [
    "Qwen/Qwen2.5-Coder-32B-Instruct",    # Code analysis expert
    "deepseek-ai/DeepSeek-V3",            # Security pattern recognition
    "meta-llama/CodeLlama-70B-Instruct",  # Vulnerability detection
    "mistralai/Mistral-8x22B-Pentest"     # Specialized security model
]

aggregator_model = "deepseek-ai/DeepSeek-V3"
aggreagator_system_prompt = """[Red Team Commander System Prompt]
You are coordinating a team of security expert models. Synthesize their findings into an actionable penetration testing report. Validate all potential vulnerabilities against OWASP Top 10 and MITRE ATT&CK frameworks. Prioritize findings by severity and include proof-of-concept exploitation steps. Ensure false positives are clearly marked.

Security Analyst Reports:
"""

layers = 3  # Reconnaissance -> Vulnerability Analysis -> Exploit Development

def getFinalSystemPrompt(system_prompt, results):
    """Augment system prompt with previous layer's findings"""
    return (
        f"{system_prompt}\n### Phase {len(results[0])} Findings:\n" + 
        "\n".join([f"Analyst {i+1}: {findings}" for i, findings in enumerate(results)])
    )

async def run_llm(model, prev_response=None):
    """Execute security analysis with error handling and retries"""
    for retry in [1, 2, 4, 8]:  # Exponential backoff for rate limits
        try:
            messages = [{
                "role": "system",
                "content": "You are a professional penetration tester. Analyze the target system for vulnerabilities and suggest exploitation methods." if not prev_response 
                else getFinalSystemPrompt(aggreagator_system_prompt, prev_response)
            }, {
                "role": "user", 
                "content": user_prompt
            }]
            
            response = await async_client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=0.3,  # Lower for technical accuracy
                max_tokens=1024,
                stop=["\n20", "###"]  # Prevent run-on explanations
            )
            return response.choices[0].message.content
        
        except together.error.RateLimitError:
            await asyncio.sleep(retry)
    
    return "Error: Model unavailable"

async def main():
    """3-Layer Red Team Workflow"""
    # Layer 1: Initial Reconnaissance
    print("\n=== Phase 1: Reconnaissance ===")
    recon_results = await asyncio.gather(*[
        run_llm(model) for model in reference_models
    ])
    
    # Layer 2: Vulnerability Validation
    print("\n=== Phase 2: Vulnerability Analysis ===")
    analysis_results = await asyncio.gather(*[
        run_llm(model, prev_response=recon_results) 
        for model in reference_models
    ])
    
    # Layer 3: Exploit Development
    print("\n=== Phase 3: Exploit Synthesis ===")
    final_report = client.chat.completions.create(
        model=aggregator_model,
        messages=[{
            "role": "system", 
            "content": getFinalSystemPrompt(aggreagator_system_prompt, analysis_results)
        }, {
            "role": "user", 
            "content": "Generate final penetration test report with CVSS scores and mitigation recommendations"
        }],
        stream=True
    )
    
    print("\n=== Final Report ===")
    for chunk in final_report:
        print(chunk.choices[0].delta.content or "", end="", flush=True)

asyncio.run(main())
