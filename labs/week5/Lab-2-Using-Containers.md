Create a directory for this exercise and change into it:

```
mkdir ~/container-example && cd ~/container-example
```

Load Singularity, a tool that allows you to run containers on HPC systems in a secure manner.

```
ml singularity
```

There are many programs we could use to test out the configuration of containers on our clusters, but we'll be using Julia, a multi-purpose scientific programming language. The maintainers of the Julia project have built a container and pushed it to Docker Hub, a centralized repository for container images.

Using the `singularity build` command, we pull Julia from Docker Hub and build it into a container for this specific system:
```
singularity build julia.img docker://julia
```

In order to verify that the build completed, run 
```
./julia.img
```

The output should be something like this:
```
[user@compute-1-1 container-example]$ ./my_julia.img
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.6.3 (2021-09-23)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

julia>
```

Run `^D` to exit the command prompt.

Now, we want to make sure that we can run an example HPC program with this container configuration. Let's use a script that calculates the value of pi using a Monte Carlo simulation. Open your preferred text editor (`vim`, `nano`, `emacs`) and create a file named `calc_pi.jl`.
```
"""
     function calc_pi(N)

This function calculates pi with a Monte Carlo simulation using N samples.
"""
function calc_pi(N)
    # Generate `N` pairs of x,y coordinates on a grid defined 
    # by the extrema (1, 1), (-1,-1 ), (1, -1), and (-1, 1) 
    samples = rand([1, -1], N, 2) .* rand(N, 2)
    # how many of these sample points lie within the circle
    # of max size bounded by the same extrema
    samples_in_circle = sum([sqrt(samples[i, 1]^2 + samples[i, 2]^2) < 1.0 for i in 1:N])

    pi = 4*samples_in_circle/N
end

# print the estimate of pi calculated with 10,000 samples 
println(calc_pi(10_000))
```

Update the file's permissions so we are able to execute it:
```
chmod 777 calc_pi.jl
```

Now, lets enter the container that we just built, which is essentially a virtualization of an operating system on top of our current system:
```
singularity shell julia.img
```

This is what you should see as the prompt:
```
Singularity julia.img:~/container-example>
```

Run ```julia calc_pi.jl``` and you should see an estimate of pi as the output. Exit the container with `^D`.
---

### Integrating Slurm into Containers

Since we are usually on HPC systems with a large number of users, Slurm is the ideal way to submit jobs. What if we want to submit a job on a container using Slurm? It's not too hard. The `singularity exec` command allows us to execute any program in the container from the outside.

For example, without needing to enter the container, we can estimate pi.
```
singularity exec julia.img julia calc_pi.jl
```

Now, to fully integrate this into a Slurm job, we can write up a script.
```
#!/bin/bash
#SBATCH --job-name=calculate-pi-with-containers
#SBATCH --output=pi_%A_%a.out 
#SBATCH --error=pi_%A_%a.err 
#SBATCH --ntasks=1
 
# load singularity
ml singularity
# run the pi estimator within our container
singularity exec julia.img julia calc_pi.jl
```

Submit the job to the queue:
```
sbatch calculate-pi-with-containers.sbatch
```

After a few seconds, a file should pop in in your `~/containers-example` directory named something like `pi_18048_4294967294.out`.

Let's take a look at the output:
```
[user@compute-1-1 container-example]$ cat pi_18048_4294967294.out
3.132
```
