Partitions for both / (root) and /boot are needed on each IM. No partition is required for swap space. The remaining disk should be a single partition for LVM.

    /boot should be ext2 on its own partition; do not use LVM.
    / (root) and remaining disk can be ext4; okay to use LVM
    Swap partitions are not required, but note that the default Ubuntu install will add a swap partition up to two times the memory.

For example, given a IM host machine with 400 GB SSD, we partition the disk to allocate a 20 GB volume for the OS and the rest to the IM LVM pool. The OS partitioning is done during the OS install and before you run the provision script; the IM LVM partitioning is done within the script.