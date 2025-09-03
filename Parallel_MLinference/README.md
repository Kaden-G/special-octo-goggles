# Parallel ML Inference with Slurm — MVP (Docker)

This is a tiny, local-first starter you can run on your laptop to simulate **parallel inference jobs with Slurm**.  
It uses a prebuilt Slurm-in-Docker image via `docker compose` to spin up a controller and two worker nodes.  
The "ML inference" here is intentionally simple (pure Python, no heavy deps) so you can verify the Slurm workflow
without long builds.

## What you get
- A Slurm controller and two compute nodes (via Docker) with a shared volume mounted at `/shared`.
- A sample **job array** that fans out lightweight inference tasks over input files in `data/inputs/`.
- Zero heavy ML deps — the demo inference uses pure Python math to keep it portable and fast.
- Clear, minimal commands you can script or expand later (add NumPy/ONNX/Torch if you want).

## Requirements
- Docker Desktop (or Docker Engine) and Docker Compose v2+
- macOS/Linux/WSL is fine

## Quickstart
1. Start the cluster:
   ```bash
   docker compose up -d
   ```

2. Get a shell on the Slurm controller (named `slurmctld` in this compose file):
   ```bash
   docker compose exec slurmctld bash
   ```

3. From inside the controller container, verify Slurm is up and see nodes:
   ```bash
   sinfo
   scontrol show nodes
   ```

4. Submit the **parallel inference** job array:
   ```bash
   sbatch /shared/slurm/job_array.sbatch
   ```

5. Watch the queue:
   ```bash
   squeue
   tail -f /shared/data/outputs/job_*.log
   ```

6. When done, view outputs on your host machine in `data/outputs/`.

7. Stop the cluster:
   ```bash
   docker compose down
   ```

## How it works
- `docker-compose.yml` launches one controller (`slurmctld`) and two workers (`slurmd01`, `slurmd02`).
- A bind mount maps this repo to `/shared` inside all containers so Slurm jobs can read inputs and write outputs.
- `slurm/job_array.sbatch` submits one task **per input file** (via a Slurm job array).
- Each task runs `scripts/infer.py` against a specific JSON input, writing a result file and a per-task log.

## Customize
- To scale workers, duplicate the worker service in `docker-compose.yml` and update hostnames `slurmd03`, etc.
- To add real ML deps, create a small venv step in the job script or bake a custom image. Keep the MVP fast first.
- To test failure modes, add `time.sleep()` or random exceptions in `scripts/infer.py` and observe retries/backoff.

## Repo tasks
Common helpers:
```bash
# Submit the job array again
docker compose exec slurmctld bash -lc "sbatch /shared/slurm/job_array.sbatch"

# Show recent Slurm completion
docker compose exec slurmctld bash -lc "sacct --format=JobID,JobName,State,Elapsed,AllocCPUS -X --starttime now-1hour"

# Stream logs
docker compose exec slurmctld bash -lc "tail -n +1 -f /shared/data/outputs/job_*.log"
```

## Notes
- This MVP uses a common Slurm-in-Docker image to keep setup minimal. If your environment blocks public pulls, swap to a pre-approved base image and install Slurm in a custom Dockerfile.
- For a more realistic pipeline later, plug in ONNX Runtime or PyTorch and batch real data. The Slurm side won’t change much — just the job payload.

---
MIT License. Have fun!
