# ofdm-banded-equalizer

A low-complexity OFDM physical-layer receiver in Python. It handles frequency-selective multipath fading and mobility-induced inter-carrier interference (ICI) using classical DSP only:

- delay-domain noise filtering for channel estimation
- a banded-matrix MMSE equalizer solved with LDL^H factorization, which cuts the equalization cost from O(N^3) to O(N Q^2)

Everything is in one notebook, `ofdm_pipeline.ipynb`.

## Pipeline

| Step | What it does | Reference idea |
|------|--------------|----------------|
| 1 | QPSK/16-QAM modem, IFFT/FFT, cyclic prefix, Rayleigh multipath + Doppler + AWGN channel | baseband engine |
| 2 | Comb pilots, LS estimate (Y_p / X_p), linear/spline interpolation, 1-tap ZF | Coleri et al. |
| 3 | Delay-domain filtering: IDFT, keep the first taps, DFT | Edfors et al. |
| 4 | High Doppler: ICI floor and CE-BEM channel estimation from pilot clusters | Al-Naffouri et al. |
| 5 | Banded MMSE equalizer using LDL^H factorization | Rugini et al. |

Model used throughout: `Y = Hf X + n`. `Hf` is diagonal for a static channel and banded (half-width Q) under Doppler.

## Setup

```bash
git clone <your-repo-url>
cd ofdm-banded-equalizer
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook ofdm_pipeline.ipynb
```

Run the cells top to bottom: definitions first, then one experiment cell per step. The full notebook takes roughly 20 seconds on a laptop.

## Parameters (first code cell)

| Name | Default | Meaning |
|------|---------|---------|
| `N` | 256 | subcarriers |
| `CP` | 32 | cyclic prefix length |
| `BITS_PER_SYM` | 2 | 2 = QPSK, 4 = 16-QAM |
| `EPS` | 0.2 | normalized Doppler (f_d / subcarrier spacing) |
| `Q` | 2 | band half-width / BEM order |
| `P_CLUSTERS` | 16 | number of pilot clusters |

## Results (default settings, QPSK)

- **ICI floor:** with perfect diagonal CSI, 1-tap ZF flattens at BER of about 2.8e-2 for eps = 0.2, about 9e-3 for eps = 0.1, and about 2.4e-3 for eps = 0.05.
- **Banded vs. dense MMSE:** identical BER, with solutions matching to about 5e-13. Banding H is the only approximation.
- **Floor reduction:** at eps = 0.2 and 40 dB, banded LDL^H reaches about 7e-3 versus about 2.7e-2 for 1-tap, roughly 4x better. A gap to the perfect-CSI receiver (about 7e-5) remains, caused by channel-estimation error and truncation to Q.
- **Complexity (one solve):** at N = 1024 dense takes about 300 ms and banded about 9 ms. Timings are machine-dependent and noisy.

The banded solver is written in pure Python loops, so it only beats NumPy's dense solve for larger N (crossover around N = 256 to 512 on the test machine).

## Known limitations

- The BEM assumes integer Doppler bins, but the simulated channel uses continuous Doppler, so a small model mismatch remains.
- The mobile-channel BER curves use few symbols and are noisy at high SNR. Increase `n_sym` for smoother curves.
- FLOP counts, a unified 1-tap/banded switch, and the optional neural-network benchmark (Step 6) are not implemented yet.
- Steps 2-3 (static channel, comb pilots) and Steps 4-5 (mobile channel, pilot clusters) are separate experiments, not one chained pipeline.

## Project layout

```
ofdm-banded-equalizer/
├── ofdm_pipeline.ipynb   # all code and experiments
├── requirements.txt
└── README.md
```

## References

- Coleri et al., channel estimation techniques for OFDM (comb/block pilots)
- Edfors et al., OFDM channel estimation by singular value decomposition (low-rank/delay-domain filtering)
- Al-Naffouri et al., channel estimation for OFDM over fast time-varying channels (basis expansion models)
- Rugini et al., simple equalization of time-varying channels for OFDM (banded-matrix equalizers)
