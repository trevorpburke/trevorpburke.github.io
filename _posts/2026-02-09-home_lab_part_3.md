---
title: "Homelab Part 3: Physical set up"
date: 2026-02-09
categories: [Blogging, Homelab]
tags: [homelab, kubernetes, k8s, talos]

description: Third post in a series about setting up a 'homelab'
---

This post will be relatively short, but I'll share some updates and progress on the physical side of my homelab setup.

### Rack Setup
I knew that I wanted to move away from stacking these mini PCs ontop of each other underneath my living room credenza, so I first took to eBay and craigslist. Searching for 'server rack' was almost entirely resulting in 19" enterprise-grade setups. I live in a San Francisco apartment and do not have the space - aesthetically or spatially - for a 19" server rack, so I eventually stumbled upon DeskPi and purchased a [12U server cabinet](https://deskpi.com/products/deskpi-rackmate-t2-light-version-10-inch-black-12u-server-cabinet-for-network-servers-audio-and-video-equipment). This would allow me space for a network switch, patch panel, 7 mini PCs, and space for a power supply (more on that later). 

For the actual trays themselves I found a seller on Etsy, [Rackify](https://www.etsy.com/shop/RackifyUS), who sold 3d printed 10" rack mounts for my Dell OptiPlex Micro, my HP EliteDesk G3, and my exact TP-Link managed switch. These rack mounts are super sleek and fit perfectly in my DeskPi RackMate T2. 

#### Rack Images

<style>
.gallery { position: relative; max-width: 480px; margin: 2rem auto; }
.gallery img { width: 100%; display: none; }
.gallery img.active { display: block; }
.gallery button { position: absolute; top: 50%; transform: translateY(-50%); 
  background: rgba(0,0,0,0.5); color: white; border: none; 
  font-size: 2rem; padding: 1rem; cursor: pointer; }
.gallery .prev { left: 0; }
.gallery .next { right: 0; }
.caption { text-align: center; padding: 0.5rem; font-style: italic; 
  color: #666; display: none; }
.caption.active { display: block; }
.dots { text-align: center; padding: 1rem; }
.dot { display: inline-block; width: 10px; height: 10px; 
  background: #ccc; border-radius: 50%; margin: 0 5px; cursor: pointer; }
.dot.active { background: #333; }
</style>

<div class="gallery" id="homelab-gallery">
  <img src="/assets/img/homelab-physical-pics/IMG_8884.jpeg" class="active">
  <img src="/assets/img/homelab-physical-pics/IMG_8889.jpeg">
  <img src="/assets/img/homelab-physical-pics/IMG_8890.jpeg">
  <img src="/assets/img/homelab-physical-pics/IMG_8910.jpeg">
  <img src="/assets/img/homelab-physical-pics/IMG_8917.jpeg">
  <img src="/assets/img/homelab-physical-pics/IMG_8936.jpeg">
  <button class="prev" onclick="changeSlide(-1)">❮</button>
  <button class="next" onclick="changeSlide(1)">❯</button>
</div>
<div class="captions">
  <p class="caption active">Beginnings of rack setup, awaiting more mounts</p>
  <p class="caption">Sizing up network cables</p>
  <p class="caption">Terminating 14-15 RJ45 connectors</p>
  <p class="caption">Installing Talos baremetal ISOs to 6 mini PCs</p>
  <p class="caption">Pre label maker labels, showing machine IP host suffix</p>
  <p class="caption">Power distribution block installed on DIN rail</p>
</div>

<div class="dots">
  <span class="dot active" onclick="currentSlide(0)"></span>
  <span class="dot" onclick="currentSlide(1)"></span>
  <span class="dot" onclick="currentSlide(2)"></span>
</div>

<script>
let slideIndex = 0;
const slides = document.querySelectorAll('#homelab-gallery img');
const captions = document.querySelectorAll('.caption');
const dots = document.querySelectorAll('.dot');

function changeSlide(n) {
  showSlide(slideIndex += n);
}

function currentSlide(n) {
  showSlide(slideIndex = n);
}

function showSlide(n) {
  if (n >= slides.length) slideIndex = 0;
  if (n < 0) slideIndex = slides.length - 1;
  
  slides.forEach(s => s.classList.remove('active'));
  captions.forEach(c => c.classList.remove('active'));
  dots.forEach(d => d.classList.remove('active'));
  
  slides[slideIndex].classList.add('active');
  captions[slideIndex].classList.add('active');
  dots[slideIndex].classList.add('active');
}
</script>

