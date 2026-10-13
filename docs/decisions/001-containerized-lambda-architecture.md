# ADR 001: Containerized AWS Lambda Architecture

[← Main README](../../README.md) · [← Documentation Index](../README.md)

- **Status:** Accepted
- **Context:** The accounting platform processes receipt batches that fluctuate wildly based on VAT deadlines and business expense reporting cycles. Receipts uploaded by clients range from low-res smartphone snapshots to ultra-high-resolution scanner outputs (>74 million pixels). The image processing workload requires heavy native dependencies (`Pillow`, `libjpeg-turbo`, `OpenCV`), dynamic LANCZOS resampling, and rapid burst scaling without paying for idle compute during off-peak hours.
- **Decision:** Package the receipt processing microservice as a containerized AWS Lambda function using the AWS Python 3.12 base image (`public.ecr.aws/lambda/python:3.12`) provisioned with high-memory execution headroom and a 600-second execution timeout.
- **Benefits:**
  - **Memory & Compute Proportionality:** In AWS Lambda, vCPU allocation scales linearly with configured memory. Scaling memory allocation automatically maximizes dedicated multi-core compute, drastically accelerating CPU-intensive LANCZOS downsampling and multi-threaded image decoding.
  - **Decompression Bomb Headroom:** High-resolution TIFF and multi-page receipts expanded into uncompressed RGBA pixel buffers require gigabytes of temporary memory during manipulation; dedicated headroom ensures zero out-of-memory (`OOM`) crashes.
  - **Zero Idle Cost:** Off-peak hours cost $0, while burst processing during Danish quarterly tax filing (*momsindberetning*) automatically scales out across concurrent Lambda invocations.
  - **Container Packaging:** Bypasses Lambda's 250MB zip deployment limit, enabling clean inclusion of native image processing binaries and ML libraries.
- **Trade-offs:**
  - Container cold starts (~3.5–5 seconds) occur on initial scaling, mitigated by AWS Lambda provisioned concurrency or keep-warm pings during business hours.
  - Ephemeral environment lacks local disk persistence beyond ephemeral `/tmp` storage, requiring immediate flushing to S3 and PostgreSQL.
- **Alternatives:**
  - *Dedicated EC2 Instances:* High maintenance overhead, idle compute costs during nights/weekends, and complex auto-scaling group tuning for sudden SQS queue spikes.
  - *Amazon ECS / Fargate Tasks:* Slower task launch times (30–60 seconds per task) compared to Lambda's sub-second container spin-up for incoming SQS messages.
  - *Standard Zip-based Lambda:* Insufficient memory headroom for uncompressed 74M+ pixel image arrays and inability to package native system libraries cleanly within zip payload constraints.
