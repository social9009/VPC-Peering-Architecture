# AWS VPC Peering — Cross-Region Private Connectivity

## Architecture Overview

```
Your Laptop
    │  SSH (public IP)
    ▼
Internet Gateway (IGW)
    │
    ▼  VPC-A  │  Mumbai (ap-south-1)  │  10.100.0.0/16
┌─────────────────────────────────────┐
│  Public Subnet 10.100.0.0/24        │
│  ┌────────────────────────────────┐ │
│  │  EC2-A  (Bastion host)         │ │
│  │  Public IP + Private IP        │ │
│  └────────────────────────────────┘ │
│            │ SSH                    │
│  Private Subnet 10.100.11.0/24      │
│  ┌────────────────────────────────┐ │
│  │  EC2-B                         │ │
│  │  Private IP only               │ │
│  └────────────────────────────────┘ │
└──────────────────┬──────────────────┘
                   │  VPC Peering (pcx-xxxxx)
┌──────────────────▼──────────────────┐
│  VPC-B  │  N.Virginia (us-east-1)  │  10.200.0.0/16
│  Private Subnet 10.200.11.0/24      │
│  ┌────────────────────────────────┐ │
│  │  EC2-C                         │ │
│  │  Private IP only               │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
    <title>AWS VPC Peering | Inter‑Region Traffic Flow Animation</title>
    <style>
        * {
            box-sizing: border-box;
        }
        body {
            background: var(--color-bg-page);
            font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
            margin: 0;
            padding: 24px 16px;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }
        .card {
            max-width: 1100px;
            width: 100%;
            background: var(--color-bg-surface);
            border-radius: 28px;
            box-shadow: 0 20px 35px -12px rgba(0,0,0,0.2);
            padding: 20px 24px 28px 24px;
            transition: all 0.2s;
        }
        /* Light / dark adaptive (system preference) */
        :root {
            color-scheme: light dark;
            --color-bg-page: #f5f7fb;
            --color-bg-surface: #ffffff;
            --color-text-primary: #1e2a3e;
            --color-text-secondary: #334155;
            --color-border-secondary: #cbd5e1;
            --color-border-tertiary: #e2e8f0;
            --color-background-secondary: #f1f5f9;
            --color-background-info: #e6f7ff;
            --color-text-info: #0066b3;
            --color-border-info: #91d5ff;
            --color-background-success: #e8f5e9;
            --color-text-success: #2e7d32;
            --color-border-success: #a5d6a7;
            --color-background-danger: #ffebee;
            --color-text-danger: #c62828;
            --color-border-danger: #ef9a9a;
        }
        @media (prefers-color-scheme: dark) {
            :root {
                --color-bg-page: #0a0c10;
                --color-bg-surface: #1e1f2c;
                --color-text-primary: #eef2ff;
                --color-text-secondary: #cbd5e6;
                --color-border-secondary: #334155;
                --color-border-tertiary: #2d3a4a;
                --color-background-secondary: #2a2f3f;
                --color-background-info: #0f2c3b;
                --color-text-info: #7bc8f4;
                --color-border-info: #1e5a7a;
                --color-background-success: #113a1e;
                --color-text-success: #6fcf97;
                --color-border-success: #2e6b3c;
                --color-background-danger: #3c1a1f;
                --color-text-danger: #f28b82;
                --color-border-danger: #a13e3e;
            }
        }
        .header {
            display: flex;
            justify-content: space-between;
            align-items: baseline;
            flex-wrap: wrap;
            margin-bottom: 16px;
        }
        h1 {
            font-size: 1.6rem;
            font-weight: 600;
            margin: 0;
            color: var(--color-text-primary);
        }
        .badge {
            background: var(--color-background-secondary);
            padding: 6px 12px;
            border-radius: 40px;
            font-size: 0.75rem;
            font-weight: 500;
            color: var(--color-text-secondary);
        }
        hr {
            border: none;
            border-top: 1px solid var(--color-border-tertiary);
            margin: 16px 0 20px 0;
        }
        .diagram-container {
            overflow-x: auto;
            margin: 8px 0 16px 0;
            border-radius: 20px;
            background: var(--color-bg-surface);
        }
        svg {
            display: block;
            width: 100%;
            min-width: 780px;
            height: auto;
            font-family: inherit;
        }
        .controls {
            margin-top: 12px;
            display: flex;
            flex-wrap: wrap;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
        }
        .step-buttons {
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
        }
        .step-btn {
            background: var(--color-background-secondary);
            border: 0.5px solid var(--color-border-secondary);
            border-radius: 40px;
            padding: 6px 16px;
            font-size: 0.8rem;
            font-weight: 500;
            cursor: pointer;
            transition: all 0.2s ease;
            color: var(--color-text-secondary);
        }
        .step-btn.active {
            background: var(--color-background-info);
            color: var(--color-text-info);
            border-color: var(--color-border-info);
        }
        .reset-btn {
            background: transparent;
            border: 1px solid var(--color-border-secondary);
            border-radius: 40px;
            padding: 6px 20px;
            font-size: 0.75rem;
            cursor: pointer;
            transition: 0.2s;
            color: var(--color-text-secondary);
        }
        .reset-btn:hover {
            background: var(--color-background-secondary);
        }
        .status-panel {
            margin-top: 18px;
            background: var(--color-background-secondary);
            border-radius: 20px;
            padding: 14px 18px;
            border-left: 4px solid;
            transition: all 0.15s;
        }
        .status-panel.ok { border-left-color: var(--color-text-success); background: var(--color-background-success); color: var(--color-text-success); }
        .status-panel.err { border-left-color: var(--color-text-danger); background: var(--color-background-danger); color: var(--color-text-danger); }
        .status-panel.tip { border-left-color: var(--color-text-info); background: var(--color-background-info); color: var(--color-text-info); }
        .status-text {
            margin: 0;
            font-size: 0.9rem;
            line-height: 1.45;
        }
        footer {
            margin-top: 24px;
            font-size: 0.7rem;
            text-align: center;
            color: var(--color-text-secondary);
            opacity: 0.7;
        }
        .tooltip-icon {
            cursor: help;
            border-bottom: 1px dotted var(--color-text-secondary);
            margin-left: 6px;
        }
    </style>
</head>
<body>
<div class="card">
    <div class="header">
        <h1>🌍 AWS VPC Peering · Mumbai → N.Virginia</h1>
        <div class="badge">interactive animated flow</div>
    </div>
    <hr>
    <div class="diagram-container">
        <svg id="vsvg" viewBox="0 0 780 520" style="background: transparent;">
            <defs>
                <marker id="arrowhead" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="5" markerHeight="5" orient="auto">
                    <path d="M 0 1 L 8 5 L 0 9" fill="none" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/>
                </marker>
                <filter id="glow" x="-20%" y="-20%" width="140%" height="140%">
                    <feGaussianBlur in="SourceAlpha" stdDeviation="3"/>
                    <feMerge>
                        <feMergeNode in="offsetblur" />
                        <feMergeNode in="SourceGraphic" />
                    </feMerge>
                </filter>
            </defs>
            <!-- Background inter-region separator -->
            <line x1="360" y1="80" x2="360" y2="460" stroke="var(--color-border-tertiary)" stroke-dasharray="6 4" opacity="0.4"/>

            <!-- Internet + IGW -->
            <circle cx="180" cy="28" r="20" fill="none" stroke="var(--color-border-secondary)" stroke-width="1.2"/>
            <ellipse cx="180" cy="28" rx="9" ry="20" fill="none" stroke="var(--color-border-secondary)" stroke-width="0.9"/>
            <line x1="160" y1="28" x2="200" y2="28" stroke="var(--color-border-secondary)" stroke-width="0.9"/>
            <text class="ts" x="180" y="58" text-anchor="middle" font-size="10" fill="var(--color-text-secondary)">Internet</text>
            <line x1="180" y1="48" x2="180" y2="68" stroke="var(--color-border-secondary)" stroke-width="1" marker-end="url(#arrowhead)"/>

            <g id="igw-group">
                <rect x="148" y="68" width="64" height="28" rx="7" fill="var(--color-background-secondary)" stroke="var(--color-border-secondary)" stroke-width="0.8"/>
                <text x="180" y="82" text-anchor="middle" dominant-baseline="central" font-weight="600" font-size="11">IGW</text>
                <text x="180" y="96" text-anchor="middle" dominant-baseline="central" font-size="8" fill="var(--color-text-secondary)">Internet GW</text>
            </g>
            <line x1="180" y1="96" x2="180" y2="155" stroke="var(--color-border-secondary)" stroke-width="1" marker-end="url(#arrowhead)"/>

            <!-- VPC-A (Mumbai) -->
            <g class="vpc-a-box">
                <rect x="20" y="102" width="330" height="380" rx="14" fill="var(--color-background-secondary)" fill-opacity="0.3" stroke="var(--color-border-secondary)" stroke-width="0.8" stroke-dasharray="3 2"/>
                <text x="30" y="122" font-weight="700" font-size="12" fill="var(--color-text-primary)">VPC-A · Mumbai (ap-south-1)</text>
                <text x="30" y="137" font-size="10" fill="var(--color-text-secondary)">CIDR 10.100.0.0/16</text>
            </g>
            <!-- Public subnet -->
            <rect x="34" y="155" width="302" height="115" rx="8" fill="var(--color-background-secondary)" fill-opacity="0.5" stroke="var(--color-border-secondary)" stroke-width="0.5"/>
            <text x="44" y="173" font-weight="600" font-size="11" fill="var(--color-text-primary)">Public subnet</text>
            <text x="320" y="173" text-anchor="end" font-size="9" fill="var(--color-text-secondary)">10.100.0.0/24</text>

            <!-- EC2-A -->
            <g id="ec2a">
                <rect x="110" y="188" width="140" height="68" rx="8" fill="var(--color-bg-surface)" stroke="var(--color-border-secondary)" stroke-width="0.8"/>
                <text x="180" y="207" text-anchor="middle" font-weight="600" font-size="11" fill="var(--color-text-primary)">EC2-A (Bastion)</text>
                <text x="180" y="224" text-anchor="middle" font-size="9" fill="var(--color-text-secondary)">Public + Private IP</text>
                <text x="180" y="240" text-anchor="middle" font-size="9" fill="var(--color-text-secondary)">SSH from internet</text>
            </g>

            <line x1="180" y1="256" x2="180" y2="290" stroke="var(--color-border-secondary)" stroke-width="1" marker-end="url(#arrowhead)"/>
            <!-- Private subnet VPC-A -->
            <rect x="34" y="292" width="302" height="150" rx="8" fill="var(--color-background-secondary)" fill-opacity="0.5" stroke="var(--color-border-secondary)" stroke-width="0.5"/>
            <text x="44" y="310" font-weight="600" font-size="11" fill="var(--color-text-primary)">Private subnet</text>
            <text x="320" y="310" text-anchor="end" font-size="9" fill="var(--color-text-secondary)">10.100.11.0/24</text>

            <!-- EC2-B -->
            <g id="ec2b">
                <rect x="110" y="324" width="140" height="68" rx="8" fill="var(--color-bg-surface)" stroke="var(--color-border-secondary)" stroke-width="0.8"/>
                <text x="180" y="343" text-anchor="middle" font-weight="600" font-size="11" fill="var(--color-text-primary)">EC2-B</text>
                <text x="180" y="360" text-anchor="middle" font-size="9" fill="var(--color-text-secondary)">Private IP only</text>
                <text x="180" y="376" text-anchor="middle" font-size="9" fill="var(--color-text-secondary)">10.100.11.x</text>
            </g>

            <!-- VPC-B (N.Virginia) -->
            <g class="vpc-b-box">
                <rect x="420" y="102" width="330" height="380" rx="14" fill="var(--color-background-secondary)" fill-opacity="0.3" stroke="var(--color-border-secondary)" stroke-width="0.8" stroke-dasharray="3 2"/>
                <text x="430" y="122" font-weight="700" font-size="12" fill="var(--color-text-primary)">VPC-B · N.Virginia (us-east-1)</text>
                <text x="430" y="137" font-size="10" fill="var(--color-text-secondary)">CIDR 10.200.0.0/16</text>
            </g>
            <!-- Private subnet VPC-B -->
            <rect x="434" y="255" width="302" height="140" rx="8" fill="var(--color-background-secondary)" fill-opacity="0.5" stroke="var(--color-border-secondary)" stroke-width="0.5"/>
            <text x="444" y="273" font-weight="600" font-size="11" fill="var(--color-text-primary)">Private subnet</text>
            <text x="720" y="273" text-anchor="end" font-size="9" fill="var(--color-text-secondary)">10.200.11.0/24</text>

            <!-- EC2-C -->
            <g id="ec2c">
                <rect x="510" y="305" width="140" height="68" rx="8" fill="var(--color-bg-surface)" stroke="var(--color-border-secondary)" stroke-width="0.8"/>
                <text x="580" y="324" text-anchor="middle" font-weight="600" font-size="11" fill="var(--color-text-primary)">EC2-C</text>
                <text x="580" y="341" text-anchor="middle" font-size="9" fill="var(--color-text-secondary)">Private IP only</text>
                <text x="580" y="357" text-anchor="middle" font-size="9" fill="var(--color-text-secondary)">10.200.11.x</text>
            </g>

            <!-- Peering connection area -->
            <g id="peering-group">
                <circle cx="380" cy="370" r="34" fill="var(--color-bg-surface)" stroke="var(--color-border-info)" stroke-width="1.2" stroke-dasharray="5 3"/>
                <text x="380" y="364" text-anchor="middle" font-weight="600" font-size="10">VPC</text>
                <text x="380" y="379" text-anchor="middle" font-weight="600" font-size="10">Peering</text>
            </g>

            <!-- Route table displays (hidden initially) -->
            <g id="rt-a-badge" opacity="0">
                <rect x="38" y="410" width="290" height="44" rx="8" fill="var(--color-background-info)" stroke="var(--color-border-info)" stroke-width="0.7"/>
                <text x="48" y="424" font-size="9" font-weight="500" fill="var(--color-text-info)">✏️ VPC-A Private route table</text>
                <text x="48" y="440" font-size="9" fill="var(--color-text-info)">Destination 10.200.0.0/16 → peering (pcx-xxxx)</text>
            </g>
            <g id="rt-b-badge" opacity="0">
                <rect x="434" y="414" width="300" height="44" rx="8" fill="var(--color-background-info)" stroke="var(--color-border-info)" stroke-width="0.7"/>
                <text x="444" y="428" font-size="9" font-weight="500" fill="var(--color-text-info)">✏️ VPC-B Private route table</text>
                <text x="444" y="444" font-size="9" fill="var(--color-text-info)">Destination 10.100.0.0/16 → peering (pcx-xxxx)</text>
            </g>

            <!-- hidden animation paths -->
            <path id="flow-pub" d="M180,188 L180,256" fill="none" stroke="none"/>
            <path id="flow-b2b" d="M180,324 L180,392 L310,392 L380,392" fill="none" stroke="none"/>
            <path id="flow-fail" d="M250,392 L380,392" fill="none" stroke="none"/>
            <path id="flow-peered" d="M250,392 L310,392 L380,392 L440,392 L580,392" fill="none" stroke="none"/>
            <path id="flow-back" d="M580,392 L440,392 L380,392 L310,392 L180,392" fill="none" stroke="none"/>
        </svg>
    </div>

    <div class="controls">
        <div class="step-buttons" id="step-buttons"></div>
        <button class="reset-btn" id="reset-animation">⟳ Reset animation</button>
    </div>
    <div class="status-panel ok" id="statusPanel">
        <p class="status-text" id="statusMsg">⚡ Step 1 — SSH into EC2-A (bastion) from your laptop.</p>
    </div>
    <footer>💡 Click each step → animated network packets show traffic flow. Learn how VPC Peering + route tables unlock cross‑region private access.</footer>
</div>

<script>
    (function(){
        // -------- state ----------
        let activeStep = 0;
        let animationFrame = null;
        let currentInterval = null;
        let resetFlag = false;

        const stepsData = [
            { name: "① SSH → EC2-A", msg: "✅ Step 1 — SSH into EC2‑A (public IP). Traffic: Your laptop → Internet → IGW → EC2‑A (10.100.0.x). Works because EC2‑A has a public IP and security group allows SSH.", type: "ok", flowType: "pub" },
            { name: "② EC2-A → EC2-B", msg: "✅ Step 2 — From EC2‑A, SSH into EC2‑B (private IP 10.100.11.x). Same VPC routing works automatically. (Tip: use SSH agent forwarding or copy key to EC2‑A).", type: "ok", flowType: "b2b" },
            { name: "③ Ping EC2-C (fails)", msg: "❌ Step 3 — From EC2‑B, ping EC2‑C (10.200.11.x). Traffic blocked — no route to VPC-B (10.200.0.0/16) in route table, and VPC peering not active yet.", type: "err", flowType: "fail" },
            { name: "④ Create Peering + Routes", msg: "🔧 Step 4 — Create VPC Peering connection (Mumbai requester, N.Virginia accepter). Then update BOTH private route tables: add routes to remote VPC CIDRs targeting the peering connection.", type: "tip", flowType: "activate" },
            { name: "⑤ Ping EC2-C (works!)", msg: "✅ Step 5 — From EC2‑B, ping EC2‑C again. Success! Packets flow: EC2‑B → private route table → VPC peering → VPC‑B private subnet → EC2‑C. Bidirectional communication established.", type: "ok", flowType: "peered" }
        ];

        // DOM elements
        const svg = document.getElementById('vsvg');
        const statusPanel = document.getElementById('statusPanel');
        const statusMsg = document.getElementById('statusMsg');
        const resetBtn = document.getElementById('reset-animation');
        const btnContainer = document.getElementById('step-buttons');

        // References to groups
        const rtABadge = document.getElementById('rt-a-badge');
        const rtBBadge = document.getElementById('rt-b-badge');
        const peeringCircle = document.getElementById('peering-group');
        
        // ----- helper: reset all animations & hide extras -----
        function resetVisuals() {
            if (currentInterval) clearInterval(currentInterval);
            if (animationFrame) cancelAnimationFrame(animationFrame);
            // hide route badges
            if (rtABadge) rtABadge.setAttribute('opacity', '0');
            if (rtBBadge) rtBBadge.setAttribute('opacity', '0');
            // remove any floating packet circles
            const packets = document.querySelectorAll('.packet-anim');
            packets.forEach(p => p.remove());
            // reset peering highlight
            if (peeringCircle) peeringCircle.style.filter = '';
        }

        // floating packet animation
        function animatePacketAlong(pathId, color, duration = 1100, repeat = false, onComplete = null) {
            const path = document.getElementById(pathId);
            if (!path) return;
            const pathLength = path.getTotalLength();
            if (pathLength === 0) return;
            const startTime = performance.now();
            let active = true;
            const packet = document.createElementNS("http://www.w3.org/2000/svg", "circle");
            packet.setAttribute("r", "6");
            packet.setAttribute("fill", color);
            packet.setAttribute("stroke", "#fff");
            packet.setAttribute("stroke-width", "1.2");
            packet.setAttribute("class", "packet-anim");
            svg.appendChild(packet);
            
            function animateFrame(now) {
                if (!active) return;
                const elapsed = now - startTime;
                let t = Math.min(1, elapsed / duration);
                const point = path.getPointAtLength(t * pathLength);
                packet.setAttribute("cx", point.x);
                packet.setAttribute("cy", point.y);
                if (t < 1) {
                    animationFrame = requestAnimationFrame(animateFrame);
                } else {
                    if (repeat) {
                        packet.remove();
                        animatePacketAlong(pathId, color, duration, true, onComplete);
                    } else {
                        packet.remove();
                        if (onComplete) onComplete();
                    }
                }
            }
            animationFrame = requestAnimationFrame(animateFrame);
            return () => { active = false; if (packet.parentNode) packet.remove(); };
        }

        function startFlow(flowType) {
            resetVisuals();
            if (flowType === "pub") {
                // show ssh to ec2-a: just ek packet on public path? we animate along igw->ec2a
                animatePacketAlong("flow-pub", "#2E7D32", 1000, true);
            } else if (flowType === "b2b") {
                animatePacketAlong("flow-b2b", "#2E7D32", 1100, true);
            } else if (flowType === "fail") {
                // red packet stops halfway and shakes?
                const stopAnim = animatePacketAlong("flow-fail", "#C62828", 800, false, () => {
                    // add a small X marker on peering area
                    const failMark = document.createElementNS("http://www.w3.org/2000/svg", "text");
                    failMark.setAttribute("x", "370");
                    failMark.setAttribute("y", "380");
                    failMark.setAttribute("fill", "#C62828");
                    failMark.setAttribute("font-size", "18");
                    failMark.setAttribute("font-weight", "bold");
                    failMark.setAttribute("class", "packet-anim");
                    failMark.textContent = "✕";
                    svg.appendChild(failMark);
                    setTimeout(() => failMark.remove(), 700);
                });
            } else if (flowType === "activate") {
                // show route tables
                if (rtABadge) rtABadge.setAttribute('opacity', '1');
                if (rtBBadge) rtBBadge.setAttribute('opacity', '1');
                if (peeringCircle) peeringCircle.style.filter = "url(#glow)";
                // also pulse peering
                const ring = peeringCircle.querySelector('circle');
                if (ring) ring.style.animation = "ringpulse 1s ease-in-out 2";
            } else if (flowType === "peered") {
                // show route tables & succesful bidirectional animation
                if (rtABadge) rtABadge.setAttribute('opacity', '1');
                if (rtBBadge) rtBBadge.setAttribute('opacity', '1');
                animatePacketAlong("flow-peered", "#1D9E75", 1300, true);
                // secondary packet backwards? optional
                setTimeout(() => {
                    animatePacketAlong("flow-back", "#1D9E75", 1300, true);
                }, 800);
            }
        }

        function setStep(index) {
            if (index < 0 || index >= stepsData.length) return;
            activeStep = index;
            // update button active style
            document.querySelectorAll('.step-btn').forEach((btn, i) => {
                if (i === index) btn.classList.add('active');
                else btn.classList.remove('active');
            });
            const step = stepsData[index];
            statusPanel.className = `status-panel ${step.type}`;
            statusMsg.innerText = step.msg;
            startFlow(step.flowType);
        }

        // build buttons
        function buildButtons() {
            stepsData.forEach((step, idx) => {
                const btn = document.createElement('button');
                btn.className = `step-btn ${idx === 0 ? 'active' : ''}`;
                btn.innerText = step.name;
                btn.addEventListener('click', () => setStep(idx));
                btnContainer.appendChild(btn);
            });
        }

        resetBtn.addEventListener('click', () => {
            resetVisuals();
            setStep(activeStep); // restart current step animation
        });

        buildButtons();
        setStep(0);
        // optional add css keyframes for pulsing
        const style = document.createElement('style');
        style.textContent = `
            @keyframes ringpulse {
                0% { stroke-opacity: 0.3; transform: scale(1); }
                50% { stroke-opacity: 1; transform: scale(1.02); }
                100% { stroke-opacity: 0.3; transform: scale(1); }
            }
        `;
        document.head.appendChild(style);
    })();
</script>
</body>
</html>


**Goal:** From your laptop, reach EC2-C (private subnet, N.Virginia) by hopping through EC2-A → EC2-B → VPC Peering → EC2-C.

---

## Step 1 — Create VPCs

### VPC-A (Mumbai)
1. Go to **AWS Console → VPC → Your VPCs → Create VPC**
2. Region: `ap-south-1` (Mumbai)
3. Name: `VPC-A`
4. IPv4 CIDR: `10.100.0.0/16`
5. Create Internet Gateway → Name: `VPC-A-IGW`
6. Attach IGW to VPC-A (Actions → Attach to VPC)

### VPC-B (N.Virginia)
1. Switch region to `us-east-1` (N.Virginia)
2. Go to **VPC → Create VPC**
3. Name: `VPC-B`
4. IPv4 CIDR: `10.200.0.0/16`
5. **No internet gateway needed** for VPC-B

> **Why non-overlapping CIDRs?** VPC peering requires that both VPCs have completely non-overlapping IP ranges. 10.100.x.x and 10.200.x.x never overlap.

---

## Step 2 — Create Subnets & Route Tables

### VPC-A (Mumbai) — 2 subnets

**Public Subnet**
| Field | Value |
|-------|-------|
| Name | `VPC-A-Public-Subnet` |
| AZ | `ap-south-1a` |
| CIDR | `10.100.0.0/24` |

Create route table `VPC-A-Public-RT`:
- Route: `0.0.0.0/0` → `IGW` (the internet gateway)
- Associate with `VPC-A-Public-Subnet`

**Private Subnet**
| Field | Value |
|-------|-------|
| Name | `VPC-A-Private-Subnet` |
| AZ | `ap-south-1a` |
| CIDR | `10.100.11.0/24` |

Create route table `VPC-A-Private-RT`:
- Default route only: `10.100.0.0/16` → `local`
- Associate with `VPC-A-Private-Subnet`

### VPC-B (N.Virginia) — 1 subnet

**Private Subnet**
| Field | Value |
|-------|-------|
| Name | `VPC-B-Private-Subnet` |
| AZ | `us-east-1a` |
| CIDR | `10.200.11.0/24` |

Create route table `VPC-B-Private-RT`:
- Default route only: `10.200.0.0/16` → `local`
- Associate with `VPC-B-Private-Subnet`

---

## Step 3 — Launch EC2 Instances

### EC2-A (VPC-A, Public Subnet — Bastion Host)

1. Launch in **Mumbai region**, `VPC-A-Public-Subnet`
2. **Enable** Auto-assign Public IP
3. Security Group: `EC2-A-SG`
   - Inbound: SSH (22) from `MyIP` or `0.0.0.0/0`
   - Outbound: All traffic
4. Key pair: `my-key.pem` (same key will be used for all 3 instances)

### EC2-B (VPC-A, Private Subnet)

1. Launch in **Mumbai region**, `VPC-A-Private-Subnet`
2. **Disable** Auto-assign Public IP
3. Security Group: `EC2-B-SG`
   - Inbound: SSH (22) from `10.100.0.0/16` (VPC-A CIDR)
   - Outbound: All traffic

### EC2-C (VPC-B, Private Subnet)

1. Launch in **N.Virginia region**, `VPC-B-Private-Subnet`
2. **Disable** Auto-assign Public IP
3. Security Group: `EC2-C-SG`
   - Inbound: SSH (22) from `10.100.0.0/16` (VPC-A CIDR)
   - Inbound: ICMP - All (ping) from `10.100.0.0/16` (VPC-A CIDR)
   - Outbound: All traffic

> **Why allow from VPC-A CIDR on EC2-C?** After peering, traffic from EC2-B will arrive at EC2-C with the source IP from the 10.100.x.x range. EC2-C's security group must allow it.

---

## Step 4 — Test Connectivity (Should Fail)

### Copy your SSH key to EC2-A
```bash
# Option A: Copy key to EC2-A
scp -i my-key.pem my-key.pem ec2-user@<EC2-A-Public-IP>:~/.ssh/

# Option B: Use SSH agent forwarding (more secure)
ssh-add my-key.pem
ssh -A -i my-key.pem ec2-user@<EC2-A-Public-IP>
```

### SSH chain: Laptop → EC2-A → EC2-B → try ping EC2-C
```bash
# From your laptop, SSH to EC2-A
ssh -i my-key.pem ec2-user@<EC2-A-Public-IP>

# From EC2-A, SSH to EC2-B (using private IP)
ssh -i ~/.ssh/my-key.pem ec2-user@10.100.11.<x>

# From EC2-B, try to ping EC2-C
ping 10.200.11.<x>
# Expected result: ping hangs / no response — NO ROUTE TO HOST
```

**Why does this fail?**
- EC2-B is in VPC-A's private subnet
- EC2-C is in VPC-B's private subnet
- The two VPCs have no connectivity — they are completely isolated by default
- No route exists in `VPC-A-Private-RT` for the `10.200.0.0/16` range

---

## Step 5 — Create VPC Peering Connection

### 5a. Copy VPC-B's ID
1. Go to **N.Virginia → VPC → Your VPCs**
2. Find VPC-B and copy its VPC ID (e.g. `vpc-0abc123def456`)

### 5b. Create the peering request (from Mumbai)
1. Go to **Mumbai → VPC → Peering Connections → Create peering connection**
2. Fill in the form:

| Field | Value |
|-------|-------|
| Name | `VPC-A-VPC-B-Peering` |
| Local VPC to peer with | Select `VPC-A` |
| Account | My account |
| Region | Another Region |
| Select region | `us-east-1` (N.Virginia) |
| VPC ID (Accepter) | Paste `vpc-0abc123def456` |

3. Click **Create Peering Connection**
4. Note the peering connection ID: `pcx-xxxxx`

### 5c. Accept the peering request (from N.Virginia)
1. Switch to **N.Virginia region**
2. Go to **VPC → Peering Connections**
3. Find the pending peering request
4. **Actions → Accept Request → Accept**

### 5d. Verify peering status
- Status should now show **Active**

> **Important:** VPC peering alone is NOT enough. You still need to update route tables — the two VPCs know about the connection but don't yet know how to route traffic through it.

---

## Step 6 — Update Route Tables

### VPC-A-Private-RT (Mumbai)
1. Go to **Mumbai → VPC → Route Tables**
2. Select `VPC-A-Private-RT` → Routes → Edit routes → Add route:

| Destination | Target |
|-------------|--------|
| `10.200.0.0/16` | `pcx-xxxxx` (select Peering Connection) |

3. Save changes

### VPC-B-Private-RT (N.Virginia)
1. Go to **N.Virginia → VPC → Route Tables**
2. Select `VPC-B-Private-RT` → Routes → Edit routes → Add route:

| Destination | Target |
|-------------|--------|
| `10.100.0.0/16` | `pcx-xxxxx` (select Peering Connection) |

3. Save changes

> **Why both route tables?** Peering is non-transitive and bidirectional — each VPC must explicitly know how to reach the other. Without both routes, traffic can leave one VPC but the response has no return path.

---

## Step 7 — Verify Full Connectivity

```bash
# From EC2-B terminal, ping EC2-C
ping 10.200.11.<x>
# Expected: PING working! You see replies

# Optional: SSH to EC2-C directly from EC2-B
ssh -i ~/.ssh/my-key.pem ec2-user@10.200.11.<x>
# Expected: Logged in to EC2-C
```

### Complete traffic path (end-to-end)

```
Your Laptop
  │  SSH (port 22, public internet)
  ▼
Internet Gateway (VPC-A)
  │
  ▼
EC2-A  10.100.0.x  (public subnet)
  │  SSH (private network, VPC-A)
  ▼
EC2-B  10.100.11.x  (private subnet, VPC-A)
  │  ICMP/SSH via VPC-A-Private-RT → pcx-xxxxx
  ▼
VPC Peering Connection
  │  via VPC-B-Private-RT → local
  ▼
EC2-C  10.200.11.x  (private subnet, VPC-B)
```

---

## Security Group Summary

| Instance | SG Name | Rule | Source | Reason |
|----------|---------|------|--------|--------|
| EC2-A | EC2-A-SG | SSH (22) inbound | `0.0.0.0/0` or My IP | Accessible from internet |
| EC2-B | EC2-B-SG | SSH (22) inbound | `10.100.0.0/16` | Only from VPC-A |
| EC2-C | EC2-C-SG | SSH (22) inbound | `10.100.0.0/16` | Only from VPC-A (via peering) |
| EC2-C | EC2-C-SG | ICMP All inbound | `10.100.0.0/16` | Allow ping from VPC-A |

---

## Route Table Summary

### VPC-A-Public-RT
| Destination | Target | Note |
|-------------|--------|------|
| `10.100.0.0/16` | local | Within VPC-A |
| `0.0.0.0/0` | `igw-xxxxx` | Internet access |

### VPC-A-Private-RT (after peering)
| Destination | Target | Note |
|-------------|--------|------|
| `10.100.0.0/16` | local | Within VPC-A |
| `10.200.0.0/16` | `pcx-xxxxx` | Route to VPC-B via peering |

### VPC-B-Private-RT (after peering)
| Destination | Target | Note |
|-------------|--------|------|
| `10.200.0.0/16` | local | Within VPC-B |
| `10.100.0.0/16` | `pcx-xxxxx` | Route to VPC-A via peering |

---

## Common Pitfalls & Troubleshooting

**Ping still not working after peering?**
- Check security group on EC2-C allows ICMP from `10.100.0.0/16`
- Confirm BOTH route tables are updated (not just one)
- Verify the peering connection status is **Active** (not pending)
- Make sure you're using EC2-C's private IP, not a public IP

**SSH to EC2-B fails from EC2-A?**
- Ensure your `.pem` key is on EC2-A or use SSH agent forwarding (`-A` flag)
- Confirm EC2-B's security group allows SSH from `10.100.0.0/16` (not just a specific IP)

**VPC peering request not visible in N.Virginia?**
- Make sure you switched to the correct region in the AWS console
- Check under VPC → Peering Connections — filter by "Pending Acceptance"

**Cannot create peering — CIDR overlap error?**
- VPC-A (10.100.0.0/16) and VPC-B (10.200.0.0/16) must not overlap
- If you accidentally used the same CIDR (e.g., 10.0.0.0/16 for both), you must delete and recreate one VPC with a different CIDR

---

## Clean-Up

### If continuing to the next exercise
1. Terminate EC2-C in VPC-B
2. Delete VPC peering connection
3. Delete VPC-B and its subnet/route table

### If stopping completely
1. Terminate all three EC2 instances (EC2-A, EC2-B, EC2-C)
2. Delete VPC peering connection
3. Delete VPC-B (N.Virginia)
4. Delete VPC-A and its subnets, route tables, and IGW (Mumbai)

> **Cost note:** Stopped EC2 instances still incur EBS storage charges. VPC peering has a per-GB data transfer cost for cross-region traffic. VPCs and subnets themselves have no ongoing cost.

---

## Key Concepts Recap

| Concept | What it means |
|---------|---------------|
| **VPC Peering** | Private network link between two VPCs (same or different account/region) |
| **Non-transitive** | Peering A↔B and B↔C does NOT give A↔C connectivity |
| **Route table required** | Peering alone does nothing — you must add routes in both VPCs |
| **Security groups** | Must allow traffic from the peer VPC's CIDR, not just the local CIDR |
| **No overlapping CIDRs** | Peered VPCs must have entirely different IP ranges |
| **Bastion / Jump host** | EC2-A acts as a gateway into the private network from the internet |
