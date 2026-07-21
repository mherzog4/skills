---
name: volley
description: Play intellectual tennis with an idea. A dynamic back-and-forth exchange where Claude and the user alternately explore and test a concept from different angles, using Socratic questioning to dig deeper, challenge assumptions, and think through edge cases together. Triggers when the user explicitly invokes it by saying "volley", "/volley", "let's volley", "volley this idea", or any phrase indicating they want to intellectually spar through rapid back-and-forth exchange. Can start from a blank slate (Claude suggests a topic) or from an existing idea the user brings. Outputs a structured exchange tracking each volley, the key insights uncovered, and a refined idea after the match.
---

# Volley: Intellectual Tennis for Ideas

Play intellectual tennis with any idea. You pitch a concept, Claude asks probing questions to explore its edges and test its assumptions. You respond, dig deeper, and volley back. The two of you alternate volleys as thinking partners, stress testing assumptions and refining understanding together until the idea is hardened or you realize it needs rethinking.

Unlike casual brainstorming, volley is rigorous by design. Concrete, grounded questions. No hand-waving. No "that's interesting, let's explore it." Every volley digs into specifics and forces real thinking. But it's collaborative. You're not being attacked. You're being challenged to think harder by a partner who genuinely wants to understand your idea better.

## How Volley Works

### Setup: Pick Your Idea

**With an existing idea**: You start by stating what you want to volley. Be specific. "A subscription model for open source" works. "Making things better" does not.

**Blank slate**: If you don't have an idea yet, ask Claude to propose something to volley about. Claude will suggest a concrete topic (a product concept, a policy, a design pattern, a business move, etc.) and you'll judge whether to run with it.

### The Match: Alternating Rounds

A volley match has two phases that repeat:

1. **Claude explores** (Volley 1, 3, 5, ...): Claude asks Socratic questions to probe the idea from a different angle. Where are the edges? What assumptions are hidden? What happens when you stress test this in the real world? Claude is genuinely curious, not trying to prove you wrong. The questions should make you think harder and see your own idea more clearly.

2. **You explore back** (Volley 2, 4, 6, ...): You respond to the questions. Answer them directly. Ask your own questions. Push back if Claude's concern misses something. Refine your thinking based on what you discover. The idea evolves as both of you learn more about it together.

### How It Ends

The match ends when either:
- You decide you're done (you've tested enough, or the idea is broken and you're ready to move on).
- Claude calls it because the idea has been thoroughly refined and only nitpicks remain.

### The Final Output

When the volley ends, you get:

1. **Match transcript**: Each volley in sequence, showing the back-and-forth clearly.
2. **Key insights uncovered**: The 3-5 biggest questions or assumptions that came up during the volley. These are the things you should think more deeply about as you build.
3. **Refined idea**: A concrete summary of what your idea is NOW, after being tested through dialogue. What became clearer? What assumptions did you have to question? What new directions emerged?

## What Makes a Good Volley

**Curious and concrete, not confrontational**
- Weak: "This won't scale."
- Strong: "At 100K users, what happens to your database query time? I'm thinking it might go from 50ms to 8 seconds. Does that match your model?"

**Grounded in reality, asked as exploration**
- Weak: "People might not want this."
- Strong: "Your target is mid-market SaaS CFOs. They probably already use three tools for this workflow. How does adoption actually work? What's the switching cost they'd face?"

**Tests assumptions through questions**
- Weak: "Your pricing might be wrong."
- Strong: "Help me understand your economics. At $50/month, what margins do you need? And what margins do your competitors actually have? I'm curious whether that price point leaves room for you to compete."

**Explores edges together**
- Weak: "What about international users?"
- Strong: "You mentioned expanding to India. What would your minimum viable price be there? And then, what do payment processing costs look like at that volume? I'm trying to understand which geographies actually work for your model."

**Invites real thinking**

The best volley is a question that makes you think alongside Claude, not against Claude. You're partners exploring the idea's shape together.

## Tips for Getting the Most Out of Volley

1. **Bring specificity**: The more concrete your idea, the better Claude can attack it. "A mobile app for X" invites weak volleys. "A mobile app where teenagers trade digital collectibles secured by blockchain, monetized through 3% per-transaction fees" invites sharp ones.

2. **Don't accept hand-waving back**: If Claude's criticism is vague, push back. "That's abstract. Give me a concrete scenario where this breaks."

3. **Actually defend**: Don't fold on the first hit. Either find a counter-argument or acknowledge the flaw and pivot the idea. The volley is most useful when you genuinely stress test both sides.

4. **Know when to stop**: After 5-8 volleys, most ideas are either significantly refined or revealed to be half-baked. You don't need to play until one side is destroyed.

5. **Save the output**: The refined idea and strongest criticisms are your product spec's shadow. Use them to de-risk the real thing.

## Example

**User**: Let's volley a SaaS product that helps restaurants automate scheduling.

**Claude (Volley 1)**: I'm curious about your assumption that automation is what restaurants actually want. Most smaller restaurants under 20 people seem to manage with WhatsApp or paper. The manager makes the schedule Sunday morning, sends it out, done in 10 minutes. Your software would take maybe 30 minutes to learn, needs data upfront, costs $40/month. What's the actual pain point that makes them switch? Or is your real wedge somewhere else?

**User (Volley 2)**: Good point on tiny places. We're going after 50-500 person restaurants where the real chaos lives. A manager spends 4 hours a week managing last-minute callouts, coverage gaps, swaps. That's where the pain is. But I'm realizing I haven't thought deeply about how we actually reach those restaurants or get them to adopt. How do you think about that wedge?

**Claude (Volley 3)**: That pain point makes sense. But here's what I'm wondering. You're competing with Toast, MarginEdge, maybe others who already have POS systems installed. A restaurant using Toast already gets basic scheduling with their POS. If they want better scheduling, what makes them choose a new tool over just asking Toast to improve what's built in? How do you think about winning against integrated competitors?

**User (Volley 4)**: That's exactly the question I should be asking. Toast's scheduling is surface level. They don't do shift swaps, open scheduling, predictive staffing. We're way better at scheduling. But you're right that Toast could copy us once they notice the demand. So maybe our moat isn't the features themselves. Maybe it's data and learning. The more scheduling data we see, the better our predictive models get. But I haven't modeled that out. Do you think that's actually defensible?

... and so on.

---

To start a volley, just tell Claude: `/volley [your idea]` or `Let's volley about [topic]`. If you don't have an idea, ask Claude to suggest one.
