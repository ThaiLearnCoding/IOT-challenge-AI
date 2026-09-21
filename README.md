# 🌐 SomniGuard - IOT AI Challenge

This project uses Machine Learning to analyze data from connected hardware sensors. Follow this guide to set up your isolated workspace from scratch and avoid library installation conflicts.

## 🚀 Quick Start Setup

### Prerequisites
Make sure you have downloaded and installed **[Anaconda](https://anaconda.com)** or **[Miniconda](https://anaconda.com)** on your system.

### 🛠️ Step-by-Step Installation

Open your **Terminal** (macOS/Linux) or **Anaconda Prompt** (Windows) and run the following commands sequentially:

#### 1. Create a Clean Environment
The `silabs-mltk` library requires an older version of Python. We will explicitly create a workspace using **Python 3.10**:
```bash
conda create --name iot-env python=3.10 -y
```

#### 2. Activate the Environment
Switch your terminal focus into the newly created environment:
```bash
conda activate iot-env
```

#### 3. Update Package Installer
Ensure `pip` is updated to prevent installation glitches:
```bash
python -m pip install --upgrade pip
```

#### 4. Install Project Tools and Dependencies
Install Jupyter support along with all required Machine Learning and processing libraries:
```bash
conda install ipykernel jupyter -y
pip install pandas numpy scikit-learn tensorflow silabs-mltk
```

---

## 📓 Running Your Notebooks in VS Code

1. Open **VS Code** and load your project folder.
2. Open your `.ipynb` notebook file.
3. Click the **Kernel Selection** button in the top-right corner of the window.
4. Select **`iot-env (Python 3.10.x)`** from the dropdown menu.
5. You are ready to click **Run All** cells!
