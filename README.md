# L-inquietude-des-cimes
A code to generate a living poetic piece of art
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>L'Inquiétude des Cimes</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body, html {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #0d0f12;
            display: flex;
            justify-content: center;
            align-items: center;
            cursor: none;
        }
        canvas {
            display: block;
            width: 100%;
            height: 100%;
            filter: contrast(110%) brightness(95%);
        }
        #grain {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            opacity: 0.04;
            pointer-events: none;
            z-index: 10;
            background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)'/%3E%3C/svg%3E");
        }
    </style>
</head>
<body>

    <div id="grain"></div>
    <canvas id="artistCanvas"></canvas>

    <script>
        const canvas = document.getElementById('artistCanvas');
        const ctx = canvas.getContext('2d');

        let width, height;
        let layers = [];
        let particles = [];
        let globalTime = 0;
        let targetMouse = { x: 0, y: 0 };
        let mouse = { x: 0, y: 0 };

        const PALETTE = {
            sky: ['#161920', '#1f242e', '#0d0f12'],
            ink: ['#080a0d', '#11141a', '#1b2029', '#2a313d', '#495366'],
            void: '#050608',
            accent: 'rgba(235, 94, 40, 0.15)'
        };

        function resize() {
            width = canvas.width = window.innerWidth;
            height = canvas.height = window.innerHeight;
            targetMouse.x = mouse.x = width / 2;
            targetMouse.y = mouse.y = height * 0.7;
            initLayers();
        }

        class RidgeLine {
            constructor(yBase, amplitude, speed, color, seed) {
                this.yBase = yBase;
                this.amplitude = amplitude;
                this.speed = speed;
                this.color = color;
                this.seed = seed;
                this.points = [];
            }

            update(time, mouseInfluence) {
                this.points = [];
                const step = 6; 
                for (let x = 0; x <= width + step; x += step) {
                    // Accumulation de bruits harmoniques pour un effet d'encre jetée ou de roche
                    let n1 = Math.sin(x * 0.002 + this.seed + time * this.speed);
                    let n2 = Math.cos(x * 0.007 - this.seed * 2 + time * this.speed * 1.5);
                    let n3 = Math.sin(x * 0.02 + this.seed) * 0.1; // Détails fins
                    
                    let noise = n1 * 0.6 + n2 * 0.35 + n3 * 0.05;
                    
                    // Convergence vers le centre de l'écran pour créer un abîme
                    let centerDist = Math.abs(x - width / 2) / (width / 2);
                    let depthProfile = Math.pow(centerDist, 1.5);

                    let y = this.yBase + noise * this.amplitude * depthProfile;
                    
                    // Souffle de la souris qui déplace la brume des montagnes
                    let mDist = Math.abs(x - mouse.x);
                    if (mDist < 300) {
                        let force = (1 - mDist / 300) * mouseInfluence;
                        y += Math.sin(x * 0.01 + time) * force * 15;
                    }

                    this.points.push({ x, y });
                }
            }

            draw() {
                ctx.beginPath();
                if (this.points.length === 0) return;
                
                ctx.moveTo(this.points[0].x, this.points[0].y);
                for (let i = 1; i < this.points.length; i++) {
                    ctx.lineTo(this.points[i].x, this.points[i].y);
                }
                
                ctx.lineTo(width, height);
                ctx.lineTo(0, height);
                ctx.closePath();
                ctx.fillStyle = this.color;
                ctx.fill();
            }
        }

        class AshParticle {
            constructor() {
                this.reset();
                this.y = Math.random() * height;
            }

            reset() {
                this.x = Math.random() * width;
                this.y = height + 10;
                this.vx = (Math.random() - 0.5) * 0.3;
                this.vy = -Math.random() * 0.4 - 0.1;
                this.size = Math.random() * 1.8 + 0.2;
                this.alpha = Math.random() * 0.3 + 0.05;
                this.life = 1;
                this.decay = Math.random() * 0.001 + 0.0005;
            }

            update(time) {
                this.x += this.vx + Math.sin(time * 0.01 + this.y * 0.01) * 0.1;
                this.y += this.vy;
                this.life -= this.decay;

                // Interaction douce avec le regard
                let dx = this.x - mouse.x;
                let dy = this.y - mouse.y;
                let dist = Math.sqrt(dx * dx + dy * dy);
                if (dist < 150) {
                    this.x += (dx / dist) * 0.5;
                }

                if (this.life <= 0 || this.y < -10 || this.x < 0 || this.x > width) {
                    this.reset();
                }
            }

            draw() {
                ctx.fillStyle = `rgba(220, 225, 235, ${this.alpha * this.life})`;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        function initLayers() {
            layers = [];
            const count = 7;
            for (let i = 0; i < count; i++) {
                let r = i / (count - 1);
                // Distribution non-linéaire pour accentuer la profondeur
                let yBase = height * 0.35 + Math.pow(r, 1.8) * (height * 0.55);
                let amp = 30 + Math.pow(r, 1.2) * 110;
                let speed = 0.08 * (1 - r * 0.7);
                
                // Interpolation manuelle des couleurs vers le noir absolu du premier plan
                let colorIndex = Math.min(Math.floor(r * PALETTE.ink.length), PALETTE.ink.length - 1);
                let color = PALETTE.ink[colorIndex];

                layers.push(new RidgeLine(yBase, amp, speed, color, i * 45.3));
            }

            particles = [];
            for (let i = 0; i < 60; i++) {
                particles.push(new AshParticle());
            }
        }

        function drawSkyGradient() {
            let grad = ctx.createLinearGradient(0, 0, 0, height);
            grad.addColorStop(0, PALETTE.sky[0]);
            grad.addColorStop(0.4, PALETTE.sky[1]);
            grad.addColorStop(0.8, PALETTE.sky[2]);
            ctx.fillStyle = grad;
            ctx.fillRect(0, 0, width, height);

            // Un astre lointain, voilé et mourant
            let sunY = height * 0.35;
            let sunGrad = ctx.createRadialGradient(width / 2, sunY, 10, width / 2, sunY, 250);
            sunGrad.addColorStop(0, 'rgba(240, 230, 220, 0.08)');
            sunGrad.addColorStop(0.3, 'rgba(240, 230, 220, 0.03)');
            sunGrad.addColorStop(1, 'rgba(0, 0, 0, 0)');
            ctx.fillStyle = sunGrad;
            ctx.beginPath();
            ctx.arc(width / 2, sunY, 250, 0, Math.PI * 2);
            ctx.fill();
        }

        function drawAtmosphere() {
            // Brouillard rampant entre l'abîme et le ciel
            let grad = ctx.createLinearGradient(0, height * 0.3, 0, height);
            grad.addColorStop(0, 'rgba(13, 15, 18, 0)');
            grad.addColorStop(0.4, 'rgba(22, 25, 32, 0.25)');
            grad.addColorStop(0.7, 'rgba(13, 15, 18, 0.7)');
            grad.addColorStop(1, PALETTE.void);
            ctx.fillStyle = grad;
            ctx.fillRect(0, height * 0.3, width, height * 0.7);

            // Éclat résiduel d'une cicatrice de lumière
            if (mouse.x > 0 && mouse.y > 0) {
                let lGrad = ctx.createRadialGradient(mouse.x, mouse.y, 0, mouse.x, mouse.y, 400);
                lGrad.addColorStop(0, PALETTE.accent);
                lGrad.addColorStop(1, 'rgba(0,0,0,0)');
                ctx.fillStyle = lGrad;
                ctx.fillRect(0, 0, width, height);
            }
        }

        function animate() {
            globalTime += 0.05;

            // Inertie fluide du mouvement de la souris
            mouse.x += (targetMouse.x - mouse.x) * 0.03;
            mouse.y += (targetMouse.y - mouse.y) * 0.03;

            ctx.clearRect(0, 0, width, height);

            drawSkyGradient();

            // Calcul de l'intensité du mouvement selon la position verticale du regard
            let mouseInfluence = (mouse.y / height) - 0.5;

            for (let i = 0; i < layers.length; i++) {
                layers[i].update(globalTime, mouseInfluence * (i + 1) * 0.3);
                layers[i].draw();

                // Injection de brume texturée entre certaines couches spécifiques
                if (i === 2 || i === 4) {
                    ctx.fillStyle = `rgba(15, 18, 22, ${0.15 * (i / layers.length)})`;
                    ctx.fillRect(0, layers[i].yBase - 100, width, height);
                }
            }

            drawAtmosphere();

            for (let p of particles) {
                p.update(globalTime);
                p.draw();
            }

            requestAnimationFrame(animate);
        }

        window.addEventListener('resize', resize);
        
        window.addEventListener('mousemove', (e) => {
            targetMouse.x = e.clientX;
            targetMouse.y = e.clientY;
        });

        window.addEventListener('touchmove', (e) => {
            if (e.touches.length > 0) {
                targetMouse.x = e.touches[0].clientX;
                targetMouse.y = e.touches[0].clientY;
            }
        });

        // Initialisation initiale
        resize();
        animate();
    </script>
</body>
</html>
