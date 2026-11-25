# Development and validation

## Local checks

```bash
python -m pip install .
python -m unittest discover -s tests
```

The tests cover:

- the codebook rules the encoder relies on;
- the container header, byte for byte against `at9tool` output;
- the 2 MiB size boundary;
- round trips at every bit rate, including silence and tiny input, the `0x01` superframe padding, and identical parallel and serial output;
- the MP3, FLAC, WAV and AIFF decoders, using the synthetic vectors in `tests/data`;
- loudness and true-peak reference levels, resampling and the limiter;
- the full command-line pipeline, including `param.json` updates.

## Regenerating generated files

| File | Command | Source |
| --- | --- | --- |
| `ps5at9/atrac9_tables.py` | `tools/gen_atrac9_tables.py /path/to/LibAtrac9 > ps5at9/atrac9_tables.py` | [LibAtrac9](https://github.com/Thealexbarney/LibAtrac9) source tree |
| `ps5at9/mp3_tables.py` | `tools/gen_mp3_tables.py /path/to/PDMP3/pdmp3.c /path/to/minimp3/minimp3.h > ps5at9/mp3_tables.py` | [PDMP3](https://github.com/technosaurus/PDMP3) and [minimp3](https://github.com/lieff/minimp3) |
| `tests/data/*` | `tools/make_test_vectors.sh` | Synthetic tones, encoded with FFmpeg and LAME |

## Checks against external references

These checks use tools that are not part of the project and are never run by it:

- **Sony decoder conformance.** Decode an encoded file with `at9tool -d -repeat 1 file.at9 out.wav`. Then compare it with the built-in decoder, rounded to 16 bits: `atrac9.Decoder(info.config).decode(info.data)`, skipping the 256-sample delay. They must match exactly. Include low bit rates and synthetic edge cases: silence, very short input, full-scale square waves, noise, sweeps and clicks.
- **Input decoders.** Compare MP3 and FLAC decoding with `ffmpeg -i file -f f32le -`. MP3 should match to about 120 dB SNR with the same gapless length, and FLAC should match exactly.
- **Quality.** Compare with `at9tool -e -br 192` on the same prepared WAV files using a PEAQ implementation such as EAQUAL or GstPEAQ.

When changing the bitstream writer or the container, repeat the Sony decoder conformance check. Record in the pull request whether the result was also checked on PS5 hardware.

## Releases

The build workflow tests installation and conversion on Linux, Windows, and
macOS with Python 3.9 and the current stable Python. It builds a wheel and
source distribution and tests both in fresh environments outside the checkout.

Update `ps5at9.__version__`, then push the matching `v<version>` tag. Once
the tests and GitHub Actions attestation pass, the workflow publishes the wheel,
source distribution, source ZIP, zipapp, and their SHA-256 checksums. This
workflow builds from the tagged commit.

## Verify a release

Download the release artifacts and checksum file. Use a recent GitHub CLI to
verify the wheel against the repository, workflow, tag, and source commit:

```bash
sha256sum -c SHA256SUMS.txt
repo=BlackBearRedux/ps5-at9-converter
tag=v0.1.1
source_sha=$(gh api "repos/$repo/commits/$tag" --jq .sha)
gh attestation verify ps5_at9_converter-0.1.1-py3-none-any.whl --repo "$repo" \
  --signer-workflow "$repo/.github/workflows/ci.yml" \
  --source-ref "refs/tags/$tag" --source-digest "$source_sha" --deny-self-hosted-runners
```

Use the appropriate asset filename and tag for your release. The legacy
`v0.1.0` release predates the attestation workflow.
