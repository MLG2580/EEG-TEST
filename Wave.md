//test test

import ddf.minim.*;
import ddf.minim.signals.*;
import ddf.minim.analysis.*;
import ddf.minim.effects.*;

float timeScale = 50;
static float normalScale = 50;
static float alphaScale = 100;
static int freqAvgScale = 50;
static int alphaCenter = 12;
static int alphaBandwidth = 2;
static int betaCenter = 24;
static int betaBandwidth = 2;

Minim minim;
AudioInput in;
FFT fft;
NotchFilter notch;
BandPass betaFilter;
BandPass alphaFilter;

int windowLength = 840;
int windowHeight = 500;
int FFTheight;
float scaling[] = {.00202, .002449/2, .0075502/2, .00589, .008864, .01777};
int FFTrectWidth = 6;
float scaleFreq = 1.33f;
float timeDomainAverage = 0;

boolean absoluteBadDataFlag;
boolean averageBadDataFlag;

float[][] averages;
int averageLength = 200;
int averageBins = 6;
int counter = 0;

void setup() {
  averages = new float[averageBins][averageLength];
  FFTheight = windowHeight - 200;
  size(840, 500, P2D);
  minim = new Minim(this);
  in = minim.getLineIn(Minim.MONO, 2048, 32768, 16);
  notch = new NotchFilter(60, 10, 32768);
  betaFilter = new BandPass(betaCenter/scaleFreq, betaBandwidth/scaleFreq, 32768);
  alphaFilter = new BandPass(alphaCenter/scaleFreq, alphaBandwidth/scaleFreq, 32768);
  in.addEffect(notch);
  fft = new FFT(in.bufferSize(), in.sampleRate());
  fft.window(FFT.HAMMING);
  rectMode(CORNERS);
}

void draw() {
  absoluteBadDataFlag = false;
  averageBadDataFlag = false;
  background(0);
  stroke(255);
  line(0, 100, windowLength, 100);
  drawSignalData();
  for (int i = 0; i < windowLength - 1; i++) {
    if (abs(in.left.get((i+1)*round(in.bufferSize()/windowLength))) > timeDomainAverage*4)
      averageBadDataFlag = true;
  }
  displayText();
  displayFreqAverages();
  counter++;
}

void keyPressed() {
  if (key == 'a') { in.removeEffect(betaFilter); if (!in.hasEffect(alphaFilter)) in.addEffect(alphaFilter); timeScale = alphaScale; }
  if (key == 'b') { in.removeEffect(alphaFilter); if (!in.hasEffect(betaFilter)) in.addEffect(betaFilter); timeScale = normalScale; }
  if (key == 'n') { in.removeEffect(alphaFilter); in.removeEffect(betaFilter); timeScale = normalScale; }
}

void drawSignalData() {
  fft.forward(in.left);
  timeDomainAverage = 0;
  for (int i = 0; i < windowLength - 1; i++) {
    stroke(255,255,255);
    if (abs(in.left.get(i*round(in.bufferSize()/windowLength)))*timeScale/normalScale > .95) {
      absoluteBadDataFlag = true;
      stroke(150,150,150);
    }
    line(i, 50 + in.left.get(i*round(in.bufferSize()/windowLength))*timeScale,
         i+1, 50 + in.left.get((i+1)*round(in.bufferSize()/windowLength))*timeScale);
    timeDomainAverage += abs(in.left.get(i*round(in.bufferSize()/windowLength)));
    if (i < (windowLength-1)/2) {
      if (i <= round(3/scaleFreq))                                                              { fill(0,0,250);   stroke(25,0,225); }
      if (i >= round(4/scaleFreq) && i <= round((alphaCenter-alphaBandwidth)/scaleFreq)-1)     { fill(50,0,200);  stroke(75,0,175); }
      if (i >= round((alphaCenter-alphaBandwidth)/scaleFreq) && i <= round((alphaCenter+alphaBandwidth)/scaleFreq)) { fill(100,0,150); stroke(125,0,125); }
      if (i >= round((alphaCenter+alphaBandwidth)/scaleFreq)+1 && i <= round((betaCenter-betaBandwidth)/scaleFreq)-1) { fill(150,0,100); stroke(175,0,75); }
      if (i >= round((betaCenter-betaBandwidth)/scaleFreq) && i <= round((betaCenter+betaBandwidth)/scaleFreq))     { fill(200,0,50);  stroke(225,0,25); }
      if (i >= round((betaCenter+betaBandwidth)/scaleFreq)+1 && i <= round(30/scaleFreq))      { fill(250,0,0);   stroke(255,0,10); }
      if (i >= round(32/scaleFreq))                                                             { fill(240,240,240); stroke(200,200,200); }
      rect(FFTrectWidth*i, FFTheight, FFTrectWidth*(i+1), FFTheight - fft.getBand(i)/10);
    }
  }
  timeDomainAverage = timeDomainAverage / (windowLength-1);
}

void displayText() {
  fill(255);
  text("absoluteBadDataFlag = " + absoluteBadDataFlag, windowLength-220, 120);
  text("averageBadDataFlag  = " + averageBadDataFlag,  windowLength-220, 140);
  text("alpha filter: " + in.hasEffect(alphaFilter),   windowLength-220, 160);
  text("beta filter:  " + in.hasEffect(betaFilter),    windowLength-220, 180);
  text("A=alpha  B=beta  N=none", 10, 490);
}

void displayFreqAverages() {
  for (int i = 0; i < 6; i++) {
    float avg = 0;
    int lowFreq = 0, hiFreq = 0;
    if (i==0) { lowFreq=0;  hiFreq=3;  fill(0,0,250);   stroke(25,0,225); }
    if (i==1) { lowFreq=3;  hiFreq=7;  fill(50,0,200);  stroke(75,0,175); }
    if (i==2) { lowFreq=alphaCenter-alphaBandwidth; hiFreq=alphaCenter+alphaBandwidth; fill(100,0,150); stroke(125,0,125); }
    if (i==3) { lowFreq=12; hiFreq=15; fill(150,0,100); stroke(175,0,75); }
    if (i==4) { lowFreq=betaCenter-betaBandwidth; hiFreq=betaCenter+betaBandwidth; fill(200,0,50); stroke(225,0,25); }
    if (i==5) { lowFreq=20; hiFreq=30; fill(250,0,0);   stroke(255,0,10); }
    int lowBound = round(fft.freqToIndex(lowFreq)/scaleFreq);
    int hiBound  = round(fft.freqToIndex(hiFreq)/scaleFreq);
    for (int j = lowBound; j <= hiBound; j++) avg += fft.getBand(j);
    avg /= (hiBound - lowBound + 1);
    avg *= scaling[i] * freqAvgScale;
    if (!absoluteBadDataFlag && !averageBadDataFlag) averages[i][counter%averageLength] = avg;
    float sum = 0;
    for (int k = 0; k < averageLength; k++) sum += averages[i][k];
    sum /= averageLength;
    rect(i*width/6, height, (i+1)*width/6, height-sum);
  }
}

void stop() {
  in.close();
  minim.stop();
  super.stop();
}
