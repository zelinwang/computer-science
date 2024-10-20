tar xvf gdb-10.1.tar.gz

cd gdb-10.1 

mkdir build 

sudo apt-get install texinfo libpython3-dev 

cd build  

../configure --with-pyhon=/usr/bin/python3 

make -j12