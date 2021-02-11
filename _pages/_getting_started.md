## Compiling PQUIC

During this process, we assume our base working directory is `working_dir`. First, you need to clone `picotls` which provides the TLS implementation on which PQUIC is based. It itself requires `openssl`.

{% highlight bash %}
$ sudo apt install libssl-dev  # or dnf install openssl-devel
$ cd working_dir
$ git clone https://github.com/p-quic/picotls.git
$ cd picotls
# the following instructions just come from the picotls README
$ git submodule init
$ git submodule update
$ cmake .
# cmake might fail if some dependencies are not met, so install them and retry the cmake
$ make
# once picotls is compiled, just go back to the working directory
$ cd ..
# pwd should return working_dir
{% endhighlight %}

Once `picotls` is set up, you can compile `pquic` with the following commands.

{% highlight bash %}
$ git clone https://github.com/p-quic/pquic
$ cd pquic
# install libarchive, used for the plugin exchange (on Fedora, dnf install libarchive-devel)
$ sudo apt install libarchive-dev
# fetch and compile ubpf, the virtual machine used within PQUIC
$ git submodule update --init
$ cd ubpf/vm
$ make
# make might generate warnings, but should compile
$ cd ../..
# compile michelfralloc dependency to handle memory
$ cd picoquic/michelfralloc
$ make
$ cd ../..
# also install gperftools, required in the current build process (on Fedora, dnf install gperftools)
$ sudo apt install google-perftools
$ cmake .
# if all the dependencies are present, cmake should not report any issue
$ make
{% endhighlight %}

At the end of this process, you should have generated several executables, including `picoquicdemo` which can act as both the PQUIC client and server.

## Checking that `picoquicdemo` works

Before going further, let's check that our compiled code can exchange data. Depending on the provided arguments, `picoquicdemo` either acts as a (P)QUIC client or a (P)QUIC server. Let's generate our first connection!

For convenience, let's open two different terminals. In the first one, we will launch the server as shown below.
{% highlight bash %}
$ cd working_dir/pquic
$ ./picoquicdemo
Starting PicoQUIC server on port 4443, server name = ::, just_once = 0, hrr= 0, 0 local plugins and 0 both plugins
{% endhighlight %}

Now on the other terminal, launch the client as follows.
{% highlight bash %}
$ cd working_dir/pquic
$ ./picoquicdemo ::1 4443
{% endhighlight %}

A lot of logs should be shown, but in the last lines of the client, you should see that the connection successfully completed without error. Great!

## Compiling your first plugin

Now that the base implementation works well, let's modify it a little such that it protects the exchanged data with Forward Erasure Correction. The `fec` plugin exactly does this, but first of all, we need to compile it into eBPF code.

For the compilation process, you need clang-6.0 and llc-6.0. You can install them by following the process at https://apt.llvm.org/. More recent versions might work, but in this case you need to modify the CLANG and LLC variables in the `Makefile`s in the plugin source code folders. Then, you can compile the `fec` plugin as follows.

{% highlight bash %}
$ cd working_dir/pquic
$ cd plugins/fec
$ make
# if it raises an error about clang-6.0 not found, update the CLANG and LLC variables in Makefile
{% endhighlight %}

That's it! Several objects files should have been generated. Each of them are ELF files containing the eBPF code implementing the plugin behavior. Let's plug them in PQUIC!

## Running your first plugin inside PQUIC

As in our first run of `picoquicdemo`, open two terminals. In the first one, we will launch the server with the `fec` plugin as follows.
{% highlight bash %}
$ cd working_dir/pquic
$ ./picoquicdemo -P plugins/fec/fec.plugin
Starting PicoQUIC server on port 4443, server name = ::, just_once = 0, hrr= 0, 1 local plugins and 0 both plugins
	local plugin plugins/fec/fec.plugin
{% endhighlight %}

And on the other terminal, we launch the client with the `fec` plugin as follows. Notice that here, for simpler log processing, we limit the transfer exchange to 10 KB by using the `-G` and `-4` options.
{% highlight bash %}
$ cd working_dir/pquic
$ ./picoquicdemo -4 -G 10000 -P plugins/fec/fec.plugin ::1 4443
{% endhighlight %}

A quick look at the client log shows that new frames, `SFPID FRAME`, are used over the connection. These frames are specific to the injected `fec` plugin, showing that we achieved to inject Forward Erasure Correction to this connection. Wonderful!
