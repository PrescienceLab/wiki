# Getting Started with the Cheese Cluster

* Nick, Alex, and Karl are the current cluster-wide cheese controllers.
* Each machine typically has an owner that has root/sudo privileges on the machine.
  This may not be the same person as the cluster-wide controllers.
  For example, person Foo may have sudo on jarlsberg for their research.
* If you need something for your work, see if you can compile/install it locally, use a Nix package, or use a container (Docker/Podman) environment.
  We try to keep these machines lean both to make adding new machines faster and because it makes research reproducibility easier for everyone in the long-run, since we don't have hidden additions.
* You will have a directory created in `/tank` for your user.
  `/tank/<username>` for `username`, for example.
  This is where you should store your work.
  `/tank` is shared between every machine in the cheese cluster.
* `/tank/project` is a place for people to put shared work, for any reason, but collaboration being the big one.
  Feel free to leverage that space, but do not delete anything from there unless you are the owner of that data.
  NOTE: Everyone has read/write access to `/tank/project`.
* Please do not use your home directories too much, since they reside on the machine-local storage.
  No machine has that much local storage.
  **Home directories are NOT shared between machines!!**
* `/tank/www` is an ***Internet-facing*** directory served by cheesemonger.
  You can access all files in `/tank/www` from <https://cheesemonger.cs.northwestern.edu>.
  For example, file `/tank/www/foo.svg` is available from <https://cheesemonger.cs.northwestern.edu/foo.svg>.
* Almost every machine in the cluster is unique.
  Dubliner is 4 socket, has a GPU, and has Intel Optane NVDIMMs.
  Burrata is ARM.
  Toussaint has an FPGA.
  Roquefort/jarlsberg/limburger are standard boxes.
  The Phis are wacky machines.
  Most people use Dubliner.
* Every machine has [Nix](https://nixos.org/) installed and flakes enabled.
* Every machine has [Docker](https://www.docker.com/) & [Podman](https://podman.io/) installed.
* Every machine can submit jobs to [Slurm](https://slurm.schedmd.com/) to run on other machines.
  Slurm jobs are shared with people interactively using the machine right now.
  There is documentation about how to use Slurm in `/tank` and/or in the Wiki.
