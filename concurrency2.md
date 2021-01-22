# Sharing data between threads

First note that if the data shared between threads is never modified then we don't have a problem at all. The problems start to occur when one or more threads modifies the shared data.