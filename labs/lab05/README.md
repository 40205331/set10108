# Distributed Parallelism with MPI

For the next two practicals it might be useful to work with a partner so you can get work on the distributed work we are undertaking. There is quite a bit of setup to do in this practical, so take your time and ensure everything is done correctly.

## Installing MPI

We are going to use Microsoft's HPC SDK to support our MPI work. To do this you need the following three items installed:

**NOW REPLACED BY 2016**

-   Microsoft HPC Pack 2012 Client Components
-   Microsoft HPC Pack 2012 SDK
-   Microsoft HPC Pack 2012 MS-MPI Redistributable Pack

You will need to install these on every machine you plan to use in your application. You will also need to add the relevant include and library folders to your project - these will be found in the **Program Files** folder. The library that you need to link against is called `msmpi.lib`.

## First MPI Application

Our first application will just initialise MPI, display some local information, and then shutdown.

```cpp
#include <iostream>
#include <mpi.h>

using namespace std;

int main()
{
    // Initialise MPI
    auto result = MPI_Init(nullptr, nullptr);
    // Check that we initialised correctly
    if (result != MPI_SUCCESS)
    {
        // Display error and abort
        cout << "ERROR - initialising MPI!" << endl;
        MPI_Abort(MPI_COMM_WORLD, result);
        return -1;
    }

    // Get MPI information
    int num_procs, rank, length;
    char host_name[MPI_MAX_PROCESSOR_NAME];
    MPI_Comm_size(MPI_COMM_WORLD, &num_procs);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Get_processor_name(host_name, &length);

    // Display information
    cout << "Number of processors = " << num_procs << endl;
    cout << "My rank = " << rank << endl;
    cout << "Running on = " << host_name << endl;

    // Shutdown MPI
    MPI_Finalize();

    return 0;
}
```

The methods of interest are `MPI_Init` (initialises MPI), `MPI_Comm_size` (gets the number of processes in the application), `MPI_Comm_rank` (gets the ID of this process) and `MPI_Finalize` (shuts down MPI).

At the moment you should just build this application - running an MPI application takes a bit more work.

## Running an MPI Application

You will need to open a command prompt in the directory where your built application is. Once you have done this, you can run the following command to execute the application locally in parallel:

```shell
mpiexec /np 4 "exe_name.exe"
```

Making sure to use the name of your application. The `/np` denotes the number of processes to use (here I use 4 - the number of logical cores on my machine). Running this command will give you an output similar to:

```shell
Number of processors = 4
Number of processors = 4
Number of processors = 4
My rank = 0
My rank = 2
My rank = 3
Number of processors = 4
Running on = kevin-ultrabook
Running on = kevin-ultrabook
Running on = kevin-ultrabook
My rank = 1
Running on = kevin-ultrabook
```

## Using a Remote Host

Running an MPI application in this method is all well and good, but we are not really doing any distributed parallelism. What we want to do is use one or more remote machines to do our processing. First you will need to find the IP address of the machine you want to use as the remote node. We do this using the `ipconfig` command:

`ipconfig`

This will give you the output similar to:

```shell
Wireless LAN adapter Wi-Fi:

   Connection-specific DNS Suffix  . : napier.ac.uk
   Link-local IPv6 Address . . . . . : fe80::55a3:e0ab:485:ce50%3
   IPv4 Address. . . . . . . . . . . : 146.176.135.164
   Subnet Mask . . . . . . . . . . . : 255.255.252.0
   Default Gateway . . . . . . . . . : 146.176.132.1
```

The value you want is the IPv4 Address - 146.176.135.164 above. Next you want to run the following command on the remote machine:

`smpd -d`

This is called a **Single-Program Multiple-Data** task. It will listen on the machine and wait for us to allocate a job. To do this, we run `mpiexec` with a few more commands:

`mpiexec /np 4 /host <ip-address> <application>`

We now tell MPI which host to run on (we can also define multiple hosts). You will also need to copy the application to the other machine. Running this version will give a similar output to before.

It is worth at this point to look at the different flags for the `mpiexec`. You can find these [here](https://docs.microsoft.com/en-us/powershell/high-performance-computing/mpiexec?view=hpc16-ps).

## Sending and Receiving

The next few short examples will look at the different methods for communication. First of all we will use the standard send and receive
messages.

```cpp
const unsigned int MAX_STRING = 100;

int main()
{
    int num_procs, my_rank;

    // Initialise MPI
    auto result = MPI_Init(nullptr, nullptr);
    if (result != MPI_SUCCESS)
    {
        cout << "ERROR - initialising MPI" << endl;
        MPI_Abort(MPI_COMM_WORLD, result);
        return -1;
    }

    // Get MPI Information
    MPI_Comm_size(MPI_COMM_WORLD, &num_procs);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);

    // Check if we are the main process
    if (my_rank != 0)
    {
        // Not main process - send message
        // Generate message
        stringstream buffer;
        buffer << "Greetings from process " << my_rank << " of " << num_procs << "!";
        // Get the character array from the string
        auto data = buffer.str().c_str();
        // Send to the main node
        MPI_Send((void*)data, buffer.str().length() + 1, MPI_CHAR, 0, 0, MPI_COMM_WORLD);
    }
    else
    {
        // Main process - print message
        cout << "Greetings from process " << my_rank << " of " << num_procs << "!" << endl;
        // Read in data from each worker process
        char message[MAX_STRING];
        for (int i = 1; i < num_procs; ++i)
        {
            // Receive message into buffer
            MPI_Recv(message, MAX_STRING, MPI_CHAR, i, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            // Display message
            cout << message << endl;
        }
    }

    // Shutdown MPI
    MPI_Finalize();

    return 0;
}
```

Here we are using the process rank to determine which process does what. The process with rank 0 we consider the main process, and its job is to receive messages (lines 32 to 45). Each other process will just send a message to the main process (lines 21 to 31).

Here we are using two new commands:

- `MPI_Send` requires the data to be sent, the size of data (we make sure we send an extra byte for a string -- the null terminator), the type of data, the destination (0 -- the main process), a tag (we will not be using tags), and the communicator.
- `MPI_Recv` requires a buffer to store the message, the maximum size of the buffer, the process to receive from, the tag, the communicator to use, and status conditions.

Running this application will give you an output similar to:

```shell
Greetings from process 0 of 4!
Greetings from process 1 of 4!
Greetings from process 2 of 4!
Greetings from process 3 of 4!
```

There are a number of different data types MPI can use beyond `MPI_CHAR` - the Introduction to Parallel Programming book will explain these further.

## Map-Reduce

Another approach to communication we can use is map-reduce. For this you will have to use a Monte-Carlo &pi; application:

```cpp
double local_sum, global_sum;

// Calculate local sum - use previously defined function
local_sum = monte_carlo_pi(static_cast<unsigned int>(pow(2, 24)));
// Print out local sum
cout.precision(numeric_limits<double>::digits10);
cout << my_rank << ":" << local_sum << endl;
// Reduce
MPI_Reduce(&local_sum, &global_sum, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);

// If main process display global reduced sum
if (my_rank == 0)
{
    global_sum /= 4.0;
    cout << "Pi=" << global_sum << endl;
}
```

`MPI_Reduce` takes the following values:

- The value to send (`local_sum`).
- The value to reduce into (`global_sum`).
- The number of elements in the send buffer (`1`).
- The type of the send buffer (`MPI_DOUBLE`).
- The reduction operation - we are using `MPI_SUM` to sum.
- The rank of the process that collects the reduction operation - we use rank `0` as the main process.
- The communicator used (`MPI_COMM_WORLD`).

Notice that we only use our main application to calculate &pi;.  Running this application will produce an output similar to:

```shell
2:3.14220428466797
3:3.14110040664673
0:3.14220428466797
1:3.14110040664673
Pi=3.14652345655735
```

## Scatter-Gather

Scatter-gather involves us taking an array of data and distributing it evenly amongst the processes. Gathering involves us gathering the results back again at the end.

For scatter-gather we are going to implement our vector normalization application. You will need the two helper to generate and normalize data.

```cpp
// Randomly generate vector values
void generate_data(vector<float> &data)
{
    // Create random engine
    auto millis = duration_cast<milliseconds>(system_clock::now().time_since_epoch());
    default_random_engine e(static_cast<unsigned int>(millis.count()));
    // Fill data
    for (unsigned int i = 0; i < data.size(); ++i)
        data[i] = e();
}

// Normalises 4D vectors
void normalise_vector(vector<float> &data)
{
    // Iterate through each 4-dimensional vector
    for (unsigned int i = 0; i < (data.size() / 4); ++i)
    {
        // Sum the squares of the 4 components
        float sum = 0.0f;
        for (unsigned int j = 0; j < 4; ++j)
            sum += powf(data[(i * 4) + j], 2.0f);
        // Get the square root of the result
        sum = sqrtf(sum);
        // Divide each component by sum
        for (unsigned int j = 0; j < 4; ++j)
            data[(i * 4) + j] /= sum;
    }
}
```

Our main application just needs to call scatter, normalize, gather.

```cpp
// Vector containing values to normalise
vector<float> data;
// Local storage.  Allocate enough space
vector<float> my_data(SIZE / num_procs);

// Check if main process
if (my_rank == 0)
{
    // Generate data
    data.resize(SIZE);
    generate_data(data);
}

// Scatter the data
MPI_Scatter(&data[0], SIZE / num_procs, MPI_FLOAT,  // Source
    &my_data[0], SIZE / num_procs, MPI_FLOAT,       // Destination
    0, MPI_COMM_WORLD);
// Normalise local data
normalise_vector(my_data);
// Gather the results
MPI_Gather(&my_data[0], SIZE / num_procs, MPI_FLOAT,// Source
    &data[0], SIZE / num_procs, MPI_FLOAT,          // Dest
    0, MPI_COMM_WORLD);

// Check if main process
if (my_rank == 0)
{
    // Display results - first 10
    for (unsigned int i = 0; i < 10; ++i)
    {
        cout << "<";
        for (unsigned int j = 0; j < 3; ++j)
            cout << data[(i * 4) + j] << ", ";
        cout << data[(i * 4) + 3] << ">" << endl;
    }
}
```

The `MPI_Scatter` command has the following parameters:

- The data to scatter - only relevant on the root process (`data`).
- The count of data to send to each process (`SIZE / num_procs`).
- The type of data sent (`MPI_FLOAT`).
- The memory to receive the data into on each process (`my_data`).
- The count of data to receive at each process (`SIZE / num_procs`).
- The type of data received (`MPI_FLOAT`).
- The root process (sender) (`0`).
- The communicator used (`MPI_COMM_WORLD`).

`MPI_Gather` is essentially this in reverse:

- The local data to send (`my_data`).
- The count of data to send from each process (`SIZE / num_procs`).
- The type of data sent (`MPI_FLOAT`).
- The memory to gather results into - only relevant onthe root process (`data`).
- The count of data to receive from each process (`SIZE / num_procs`).
- The type of data received (`MPI_FLOAT`).
- The root process (gatherer) (`0`).
- The communicator used (`MPI_COMM_WORLD`).

Running this application will provide the output shown:

```shell
<0.47411, 0.499097, 0.318984, 0.651438>
<0.328696, 0.597439, 0.312654, 0.661266>
<0.369751, 0.563316, 0.00549236, 0.73887>
<0.513968, 0.515516, 0.55603, 0.401137>
<0.557941, 0.572452, 0.567326, 0.197844>
<0.607652, 0.0981679, 0.781428, 0.102427>
<0.332849, 0.046342, 0.620809, 0.708279>
<0.654357, 0.100316, 0.714417, 0.226631>
<0.791911, 0.187327, 0.226341, 0.535308>
<0.301491, 0.402493, 0.653127, 0.566151>
```

You can check to see if these vectors are normalised.

## Broadcast

The final communication type we will look at is broadcast. Broadcasting just allows us to send a message from one source to all processes on the communicator.

```cpp
// Check if main process
if (my_rank == 0)
{
    // Broadcast message to workers
    string str = "Hello World!";
    MPI_Bcast((void*)&str.c_str()[0], str.length() + 1, MPI_CHAR, 0, MPI_COMM_WORLD);
}
else
{
    // Receive message from main process
    char data[100];
    MPI_Bcast(data, 100, MPI_CHAR, 0, MPI_COMM_WORLD);
    cout << my_rank << ":" << data << endl;
}
```

`MPI_Bcast` takes the following parameters:

- Data to broadcast (sender sends, receiver reads into this data) ('data`).
- Count of data to send / receive) (`100`).
- Data type sent / received (`MPI_CHAR`).
- Root node (`0`).
- Communicator (`MPI_COMM_WORLD`).

## Exercises

1.  As always you should be taking timings of your applications
2.  The Mandelbrot is quite an interesting application to distribute. In particular you will find that our implementation allows some parts to be processed quickly, and other parts slowly. You should try and divide the work so that you can optimise performance.
3.  Try and get an application that works across a number of hosts. Try four machines in the games lab (16 processes in total). Again gather timings. Mandelbrot is another good application here.
