const express = require('express');
const { Notion } = require('@neurosity/notion');
const { sliceFFT } = require('@neurosity/pipes');
const fs = require('fs');
const { combineLatest } = require('rxjs');

const app = express();
const PORT = 3001;
const thetaDataFilename = 'neurosity_middleware/ThetaData.txt';
const additionalMetricsFilename = 'neurosity_middleware/AdditionalMetrics.txt';

// Thresholds for frisson detection (will be dynamically calculated)
const dynamicWindowSize = 50; // Number of recent data points to consider

// EEG channel mapping for Notion 2 (verify if using a different device)
const channelMap = {
  0: "C3",
  1: "C4",
  2: "CP3",
  3: "CP4",
  4: "F5",
  5: "F6",
  6: "PO3",
  7: "PO4"
};

// Define ROIs with corrected mapping for RT
const ROIMapping = {
  RPF: "F6",  // Right Prefrontal
  RC: "C4",   // Right Central
  RT: "CP4"   // Right Temporal
};

// Function to get channel index by label
function getChannelIndexByLabel(label) {
  for (const [index, electrode] of Object.entries(channelMap)) {
    if (electrode === label) {
      return Number(index);
    }
  }
  return null;
}

// Precompute channel indices for each ROI
const RPFIndex = getChannelIndexByLabel(ROIMapping.RPF); // F6
const RCIndex = getChannelIndexByLabel(ROIMapping.RC);   // C4
const RTIndex = getChannelIndexByLabel(ROIMapping.RT);   // CP4

const notion = new Notion();
let isLoggedIn = false;

// Variables for dynamic thresholding
const recentThetaPower = {
    RPF: [],
    RC: [],
    RT: []
};

// Cooldown and consecutive exceedances for thirdeye detection
let frissonCooldown = false;
const cooldownTime = 1000; // in milliseconds
let consecutiveExceedances = 0;
const requiredConsecutive = 1.3;

// New variables for extended  logging
let frissonActive = false;   
let frissonTimeout = null;   
const frissonActiveDuration = 2500; // 2 seconds after detection

// Helper functions
function calculateMean(data) {
    if (data.length === 0) return 0;
    return data.reduce((sum, value) => sum + value, 0) / data.length;
}

function calculateStd(data, mean) {
    if (data.length === 0) return 0;
    return Math.sqrt(data.reduce((sum, value) => sum + Math.pow(value - mean, 2), 0) / data.length);
}

function calculateBetaToAlpha(betaData, alphaData) {
    const ratio = {};
    for (const key in betaData) {
        ratio[key] = alphaData[key] !== 0 ? betaData[key] / alphaData[key] : 0;
    }
    return ratio;
}

// Updated alpha asymmetry calculation to match the paper (simple difference)
function calculateAlphaAsymmetry(alphaDataLeft, alphaDataRight) {
    const asymmetry = {};
    for (const keyLeft in alphaDataLeft) {
        const correspondingRight = keyLeft.replace('L', 'R');
        if (alphaDataRight[correspondingRight] !== undefined) {
            const leftVal = alphaDataLeft[keyLeft];
            const rightVal = alphaDataRight[correspondingRight];
            // Paper-aligned: simple difference (Right - Left)
            asymmetry[correspondingRight] = rightVal - leftVal;
        }
    }
    return asymmetry;
}

// Track theta frisson
function trackThetaFrisson(psdData) {
    updateRecentTheta(psdData);

    const dynamicThresholds = calculateDynamicThresholds();

    // According to the paper: RPF > threshold, RC < threshold, RT < threshold
    const frissonDetected = (psdData.RPF > dynamicThresholds.RPF) &&
                            (psdData.RC < dynamicThresholds.RC) &&
                            (psdData.RT < dynamicThresholds.RT);

    if (frissonDetected) {
        consecutiveExceedances += 1;
        if (consecutiveExceedances >= requiredConsecutive && !frissonCooldown) {
            console.log("Frisson detected: High RPF theta and low RC/RT theta.");
            frissonCooldown = true;

            // Activate frisson state
            activateFrissonState();

            setTimeout(() => {
                frissonCooldown = false;
            }, cooldownTime);

            consecutiveExceedances = 0; // Reset after detection
        }
    } else {
        consecutiveExceedances = 0;
    }
}

// Activate frisson state and set/extend the timer
// Activate frisson state and set/extend the timer
async function activateFrissonState() {
    frissonActive = true;

    // Log frisson activation
    console.log("[Third Eye Closing/Opening // METACONSCIOUSNESS ACHIEVED]");

    if (frissonTimeout) {
        clearTimeout(frissonTimeout);
    }



    frissonTimeout = setTimeout(() => {
        frissonActive = false;
        console.log("Frisson no longer active.");
    }, frissonActiveDuration);
}



function updateRecentTheta(psdData) {
    for (const key in recentThetaPower) {
        recentThetaPower[key].push(psdData[key]);
        if (recentThetaPower[key].length > dynamicWindowSize) {
            recentThetaPower[key].shift();
        }
    }
}

function calculateDynamicThresholds() {
    const thresholds = {};
    for (const key in recentThetaPower) {
        const data = recentThetaPower[key];
        const mean = calculateMean(data);
        const std = calculateStd(data, mean);
        thresholds[key] = mean + 1.45 * std; // Adjust multiplier as needed
    }
    return thresholds;
}

// Extract ROI data helper function
function extractROIData(psdData, RPFIndex, RCIndex, RTIndex) {
    if (!Array.isArray(psdData)) return null;
    const channelAverages = psdData.map(channel =>
        channel.reduce((sum, val) => sum + Math.abs(val), 0) / channel.length
    );
    return {
        RPF: channelAverages[RPFIndex],
        RC: channelAverages[RCIndex],
        RT: channelAverages[RTIndex]
    };
}

// Single channel avg helper
function extractSingleChannelAvg(psdData, index) {
    if (!Array.isArray(psdData) || index == null) return 0;
    const channel = psdData[index];
    if (!Array.isArray(channel)) return 0;
    return channel.reduce((sum, val) => sum + Math.abs(val), 0) / channel.length;
}

// Function to calculate and log additional metrics
function calculateAndLogAdditionalMetrics(psdDataAlpha, psdDataBeta, timestamp) {
    // Calculate Beta to Alpha Ratio
    const betaToAlphaRatio = calculateBetaToAlpha(psdDataBeta, psdDataAlpha);

    // Log Beta to Alpha Ratio
    console.log(`[${timestamp}] Beta/Alpha Ratio - RPF: ${betaToAlphaRatio.RPF.toFixed(3)}, RC: ${betaToAlphaRatio.RC.toFixed(3)}, RT: ${betaToAlphaRatio.RT.toFixed(3)}`);

    // Calculate Alpha Asymmetry (now aligned with the paper)
    const psdDataAlphaLeft = {
        LPF: psdDataAlpha.LPF || 0,
        LC: psdDataAlpha.LC || 0,
        LT: psdDataAlpha.LT || 0
    };

    const psdDataAlphaRight = {
        RPF: psdDataAlpha.RPF,
        RC: psdDataAlpha.RC,
        RT: psdDataAlpha.RT
    };

    const alphaAsymmetry = calculateAlphaAsymmetry(psdDataAlphaLeft, psdDataAlphaRight);

    // Log Alpha Asymmetry
    for (const key in alphaAsymmetry) {
        console.log(`[${timestamp}] Alpha Asymmetry (${key} vs corresponding Left): ${alphaAsymmetry[key].toFixed(3)}`);
    }

    // Prepare Additional Metrics Log Entry
    let additionalMetricsEntry = `[${timestamp}] Beta/Alpha RPF: ${betaToAlphaRatio.RPF.toFixed(3)}, RC: ${betaToAlphaRatio.RC.toFixed(3)}, RT: ${betaToAlphaRatio.RT.toFixed(3)}, `;
    for (const key in alphaAsymmetry) {
        additionalMetricsEntry += `AlphaAsymmetry ${key}: ${alphaAsymmetry[key].toFixed(3)}, `;
    }
    // Remove trailing comma and space
    additionalMetricsEntry = additionalMetricsEntry.slice(0, -2);

    // Log Additional Metrics to file
    fs.appendFile(additionalMetricsFilename, additionalMetricsEntry + '\n', err => {
        if (err) console.error('Error saving additional metrics:', err);
    });
}

// Log Theta Data
function logThetaData() {
    if (!isLoggedIn) {
        console.error("Not logged in to Neurosity. Cannot start logging data.");
        return;
    }

    // Subscribe to theta band
    notion.brainwaves("raw").pipe(
        sliceFFT([4, 8]) // Theta frequency range
    ).subscribe(psdWrapper => {
        const psdData = psdWrapper.psd?.data;

        if (Array.isArray(psdData)) {
            // Extract ROI data for theta
            const psdDataROI = extractROIData(psdData, RPFIndex, RCIndex, RTIndex);

            if (psdDataROI) {
                const timestamp = new Date().toISOString();

                // Modify log message if frisson is active
                const frissonStatus = frissonActive ? " [Third Eye Closing/Opening // METACONSCIOUSNESS ACHIEVED]" : "";
                console.log(
                    `[${timestamp}] Theta Power - RPF (F6): ${psdDataROI.RPF.toFixed(3)}, ` +
                    `RC (C4): ${psdDataROI.RC.toFixed(3)}, RT (CP4): ${psdDataROI.RT.toFixed(3)}${frissonStatus}`
                );

                // Track frisson
                trackThetaFrisson(psdDataROI);

                // Log to file as well with frisson status
                const thetaLogEntry = `[${timestamp}] Theta RPF: ${psdDataROI.RPF.toFixed(3)}, ` +
                                      `RC: ${psdDataROI.RC.toFixed(3)}, RT: ${psdDataROI.RT.toFixed(3)}${frissonStatus}`;
                fs.appendFile(thetaDataFilename, thetaLogEntry + '\n', err => {
                    if (err) console.error('Error saving theta data:', err);
                });
            }
        }
    }, error => {
        console.error("Error in theta brainwaves stream:", error);
    }, () => {
        console.log("Theta brainwaves stream completed.");
    });
}

// Setup Alpha and Beta Streams
function setupAlphaBetaLogging() {
    const alpha$ = notion.brainwaves("raw").pipe(
        sliceFFT([8, 12]) // Alpha frequency range
    );

    const beta$ = notion.brainwaves("raw").pipe(
        sliceFFT([13, 30]) // Beta frequency range
    );

    combineLatest([alpha$, beta$]).subscribe(([alphaWrapper, betaWrapper]) => {
        const timestamp = new Date().toISOString();

        // Extract ROI data for alpha and beta
        const alphaDataROI = extractROIData(alphaWrapper.psd?.data, RPFIndex, RCIndex, RTIndex);
        const betaDataROI = extractROIData(betaWrapper.psd?.data, RPFIndex, RCIndex, RTIndex);

        // For alpha asymmetry, we need left ROI alpha data (LPF, LC, LT)
        const LPFIndex = getChannelIndexByLabel("F5"); // Left Prefrontal
        const LCIndex = getChannelIndexByLabel("C3");  // Left Central
        const LTIndex = getChannelIndexByLabel("CP3"); // Left Temporal

        const LPFAvg = extractSingleChannelAvg(alphaWrapper.psd?.data, LPFIndex);
        const LCAvg = extractSingleChannelAvg(alphaWrapper.psd?.data, LCIndex);
        const LTAvg = extractSingleChannelAvg(alphaWrapper.psd?.data, LTIndex);

        // Now we have alpha data for left and right
        const fullAlphaData = {
            RPF: alphaDataROI?.RPF || 0,
            RC: alphaDataROI?.RC || 0,
            RT: alphaDataROI?.RT || 0,
            LPF: LPFAvg,
            LC: LCAvg,
            LT: LTAvg
        };

        if (alphaDataROI && betaDataROI) {
            calculateAndLogAdditionalMetrics(fullAlphaData, betaDataROI, timestamp);
        }

    }, error => {
        console.error("Error in alpha/beta streams:", error);
    }, () => {
        console.log("Alpha/Beta streams completed.");
    });
}

// Function to login to Neurosity
async function loginToNeurosity() {
    try {
        await notion.login({
            email: "EMAIL",
            password: "PASSWORD" // Consider using environment variables for security
        });
        console.log("Logged in to Neurosity");
        isLoggedIn = true;
        logThetaData();
        setupAlphaBetaLogging();
    } catch (error) {
        console.error("Error logging in:", error);
    }
}

// Start the login process and initiate data logging once logged in
loginToNeurosity();

app.get('/', (req, res) => {
    res.send('Neurosity Middleware Running');
});

app.listen(PORT, () => {
    console.log(`Middleware listening at http://localhost:${PORT}`);
});

