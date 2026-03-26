🐟 ZebraRestorer: Deep Learning for Bio-Imaging 🧬
ZebraRestorer is a computer vision pipeline designed to fix and enhance images of zebrafish larvae. It specializes in "cleaning" images captured in extremely low-light conditions, where traditional cameras only see grainy noise. Using a U-Net neural network, the system learns how to reconstruct high-quality anatomical details from raw, noisy data.

🚀 What it does
🔬 Low-Light Recovery: Specifically tuned for "Photon Counting" data—where the camera detects individual particles of light.

✨ Noise Cleaning: Uses a specialized pre-processing step to stabilize image fluctuations, making it easier for the AI to understand the shapes.

🧠 Smart Reconstruction: Features a U-Net architecture that looks at both the "big picture" (the larva's shape) and the "tiny details" (cells and tissues) at the same time.

🛠️ Smart Data Handling: Includes a custom system to stack and align thousands of rows of raw sensor data into clear, 2D images.

📐 Reliable Testing: Uses a strict spatial-split method to ensure the AI is actually learning and not just "memorizing" the images.

🛠️ How it works (The Engineering Side)
The project is built with a modular approach:

The Pre-Processor: Cleans the raw data and prepares it for the neural network.

The Model: A "Deep Convolutional" network that acts like a smart filter, removing grain while keeping the edges sharp.

The Trainer: A robust loop that feeds small "patches" of images to the AI, using data augmentation (rotations and flips) to make the model more accurate.

The Scaler: Automatically adjusts image sizes to make sure the input and output match perfectly.
