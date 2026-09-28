# Docker and Docker Compose

Open `D510_Docker_and_Docker_Compose.ipynb` in Jupyter and read the rendered Markdown cells. Commands are shown as text for use in a terminal.

The Docker notebook links to five SVG diagrams in `assets/` using relative Markdown paths. The Spark notebook links to `assets/spark-docker.svg`. Upload the notebooks and the complete `assets/` folder together, preserving their names and directory structure, so the diagrams resolve on GitHub and in Jupyter. The ready-to-run example is in `compose-demo/compose.yaml`.

Keep `assets/` beside the notebooks when opening them.

Installation and Docker commands target a remote Ubuntu VM only, not WSL or a corporate laptop. The notebook includes apt installation, group access, verification, and SSH port forwarding for the web examples.

## Spark image and cluster

Continue with `D511_Spark_Docker_Image_and_Compose.ipynb`. Its self-contained companion project is in `spark-docker/`; follow that folder's README to build and run the master and worker.
