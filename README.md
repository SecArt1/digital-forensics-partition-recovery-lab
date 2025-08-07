# Digital Forensics Lab: Partition Recovery and File Carving
## Complete Linux Instructions - Create Your Own Lab Environment

### Lab Objectives
- **Create a realistic forensic scenario** by building your own test disk image
- **Understand partition table structure** through hands-on creation with fdisk
- **Simulate partition deletion** to create forensic challenge
- **Recover deleted partition** using TestDisk on your created image
- **Access and analyze files** inside the recovered partition
- **Perform file carving** inside the restored partition to recover missing files
- **Document findings** with proper forensic procedures
- **Compare results** with original intact image for verification

---

## Prerequisites and Setup

### System Requirements
- Linux distribution (Ubuntu 20.04+ recommended)
- Root/sudo privileges
- Minimum 4GB RAM, 20GB free disk space
- Network connection for tool installation

### Required Tools Installation

#### Install TestDisk (Partition Recovery)
```bash
# Update package repositories
sudo apt update

# Install TestDisk and PhotoRec
sudo apt install testdisk

# Verify installation
testdisk --version
photorec --version
```

#### Install Foremost (File Carving)
```bash
# Install Foremost
sudo apt install foremost

# Verify installation
foremost -V
```

#### Install Scalpel (Alternative File Carving Tool)
```bash
# Install Scalpel
sudo apt install scalpel

# Verify installation
scalpel -V
```

#### Install Autopsy (Optional - GUI Analysis)
```bash
# Add Autopsy repository
wget -q -O - https://sleuthkit.org/sleuthkit-debian.asc | sudo apt-key add -
echo "deb https://sleuthkit.org/sleuthkit-debian/ focal main" | sudo tee /etc/apt/sources.list.d/sleuthkit.list

# Update and install
sudo apt update
sudo apt install sleuthkit autopsy

# Note: For newer Ubuntu versions, use snap:
sudo snap install autopsy
```

#### Install Additional Forensic Tools
```bash
# Install hashing utilities
sudo apt install coreutils

# Install hex editor (optional)
sudo apt install hexedit bless

# Install file analysis tools
sudo apt install file binutils

# Verify hash tools
md5sum --version
sha1sum --version
sha256sum --version
```

---

## Lab Environment Setup

### Create Working Directory
```bash
# Create lab directory structure
mkdir -p ~/forensics_lab/partition_recovery
cd ~/forensics_lab/partition_recovery

# Create subdirectories for organization
mkdir -p evidence original_images recovered_partitions carved_files reports logs
```

### Create Lab Environment (Creating Your Own Test Image)


#### Create Your Own Test Image 

##### Step 1: Create a Blank Disk Image
```bash
# Navigate to evidence directory
cd ~/forensics_lab/partition_recovery/evidence

# Create a 1GB blank disk image
dd if=/dev/zero of=test_disk.dd bs=1M count=1024

# Verify the image was created
ls -lh test_disk.dd
file test_disk.dd

# Document image creation
echo "=== Test Disk Image Creation ===" > ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Image file: test_disk.dd" >> ../logs/lab_creation.log
echo "Size: 1GB (1024MB)" >> ../logs/lab_creation.log
ls -la test_disk.dd >> ../logs/lab_creation.log
```

##### Step 2: Attach Disk Image to a Loop Device
```bash
# Attach the disk image to a loop device
sudo losetup /dev/loop0 test_disk.dd

# Verify loop device attachment
sudo losetup -l | grep loop0

# Document loop device attachment
echo "=== Loop Device Attachment ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Loop device: /dev/loop0" >> ../logs/lab_creation.log
echo "Image: test_disk.dd" >> ../logs/lab_creation.log
```

##### Step 3: Create a New Partition Table and Primary Partition
```bash
# Use fdisk to create partition table and partition
sudo fdisk /dev/loop0

# In fdisk interactive mode, perform these steps:
# 1. Type 'o' to create a new empty DOS partition table
# 2. Type 'n' to create a new partition
# 3. Type 'p' for primary partition
# 4. Type '1' for partition number 1
# 5. Press Enter to accept default first sector
# 6. Press Enter to accept default last sector (uses entire disk)
# 7. Type 'w' to write the partition table and exit

# Alternative: Use fdisk non-interactively
sudo fdisk /dev/loop0 << EOF
o
n
p
1


w
EOF

# Expected Output After 'w' Command:
# The partition table has been altered.
# Calling ioctl() to re-read partition table.
# Re-reading the partition table failed.: Invalid argument
# 
# The kernel still uses the old table. The new table will be used at
# the next reboot or after you run partprobe(8) or partx(8).
#
# NOTE: This error message is NORMAL for loop devices and expected behavior.
# The partition table was successfully written - we'll fix the kernel table in Step 4.

# Verify partition creation (may not show partition yet - this is normal)
sudo fdisk -l /dev/loop0

# Document partition creation
echo "=== Partition Table Creation ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Partition table: DOS/MBR" >> ../logs/lab_creation.log
echo "Primary partition: /dev/loop0p1 (will be accessible after Step 4)" >> ../logs/lab_creation.log
echo "Expected kernel table warning: NORMAL for loop devices" >> ../logs/lab_creation.log
sudo fdisk -l /dev/loop0 >> ../logs/lab_creation.log
```

##### Step 4: Refresh Partition Table Mapping
```bash
# IMPORTANT: The error message from Step 3 is normal for loop devices.
# This step resolves the "Invalid argument" error by refreshing the partition table.

# Method 1: Use partprobe to refresh partition table
sudo partprobe /dev/loop0

# If partprobe is not available, install it:
# sudo apt install parted

# Method 2: If partprobe fails, use the detach/reattach method
# This is the most reliable method for loop devices
sudo losetup -d /dev/loop0
sudo losetup -P /dev/loop0 test_disk.dd

# Method 3: Alternative using kpartx (if available)
# sudo kpartx -a /dev/loop0

# Verify partition is now accessible
lsblk /dev/loop0
ls -la /dev/loop0*

# You should now see /dev/loop0p1 listed
# If you still don't see /dev/loop0p1, try:
sudo partx -a /dev/loop0

# Final verification
echo "Available devices:"
ls -la /dev/loop0*

# Document partition mapping
echo "=== Partition Mapping Refresh ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Method used: partprobe and loop device refresh" >> ../logs/lab_creation.log
echo "Kernel table error resolved: YES" >> ../logs/lab_creation.log
lsblk /dev/loop0 >> ../logs/lab_creation.log

# Troubleshooting note: If /dev/loop0p1 still doesn't appear:
# 1. Check if the loop device is properly attached: losetup -l
# 2. Verify the partition table was written: fdisk -l /dev/loop0
# 3. Try using a different loop device: /dev/loop1, /dev/loop2, etc.
# 4. Reboot the system as last resort (usually not necessary)
```

##### Step 5: Format the New Partition
```bash
# Format the partition with ext4 filesystem
sudo mkfs.ext4 /dev/loop0p1

# Alternative filesystem options:
# sudo mkfs.ntfs -f /dev/loop0p1        # NTFS filesystem
# sudo mkfs.fat -F 32 /dev/loop0p1      # FAT32 filesystem
# sudo mkfs.ext3 /dev/loop0p1           # ext3 filesystem

# Verify filesystem creation
sudo file -s /dev/loop0p1
sudo blkid /dev/loop0p1

# Document filesystem creation
echo "=== Filesystem Creation ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Filesystem: ext4" >> ../logs/lab_creation.log
sudo blkid /dev/loop0p1 >> ../logs/lab_creation.log
```

##### Step 6: Mount Partition and Add Files
```bash
# Create mount point
sudo mkdir -p /mnt/test_partition

# Mount the partition
sudo mount /dev/loop0p1 /mnt/test_partition

# Verify mount
df -h | grep test_partition

# Create test files and directories
sudo mkdir -p /mnt/test_partition/{documents,images,archives,evidence}

# Create sample text files
sudo bash -c 'echo "This is a test document for forensic analysis" > /mnt/test_partition/documents/case_notes.txt'
sudo bash -c 'echo "Financial transaction log: $50,000 transfer to account 12345" > /mnt/test_partition/documents/financial_log.txt'
sudo bash -c 'echo "Meeting notes: Discuss project timeline and deliverables" > /mnt/test_partition/documents/meeting_notes.txt'

# Create a sample config file
sudo bash -c 'cat > /mnt/test_partition/evidence/system_config.conf << EOF
# System Configuration File
server_ip=192.168.1.100
database_password=SecretPassword123
encryption_key=ABC123DEF456
backup_location=/srv/backups
EOF'

# Create some sample binary files (simulated)
sudo dd if=/dev/urandom of=/mnt/test_partition/evidence/encrypted_data.bin bs=1K count=50
sudo dd if=/dev/urandom of=/mnt/test_partition/images/photo001.jpg bs=1K count=500

# Create a compressed archive with evidence
cd /tmp
echo "Critical evidence file" > critical_evidence.txt
echo "Forensic analysis data" > analysis_data.txt
tar -czf evidence_archive.tar.gz critical_evidence.txt analysis_data.txt
sudo mv evidence_archive.tar.gz /mnt/test_partition/archives/
rm critical_evidence.txt analysis_data.txt

# Set realistic timestamps
sudo touch -t 202501150930 /mnt/test_partition/documents/case_notes.txt
sudo touch -t 202501200845 /mnt/test_partition/documents/financial_log.txt
sudo touch -t 202502010600 /mnt/test_partition/evidence/system_config.conf

# List created files
echo "=== Test Files Created ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
sudo find /mnt/test_partition -type f -exec ls -la {} \; >> ../logs/lab_creation.log

# Calculate hashes of test files for verification
echo "=== Test File Hashes ===" >> ../logs/lab_creation.log
sudo find /mnt/test_partition -type f -exec md5sum {} \; >> ../logs/lab_creation.log
```

##### Step 7: Delete Partition Table (Simulate Partition Deletion)
```bash
# First, unmount the partition
sudo umount /mnt/test_partition
sudo rmdir /mnt/test_partition

# Create a backup of the current state (with partition table)
cp test_disk.dd test_disk_with_partition.dd

# Now simulate partition deletion by zeroing the partition table
# The partition table is stored in the first 512 bytes (Master Boot Record)
dd if=/dev/zero of=test_disk.dd bs=512 count=1 conv=notrunc

# Alternative: Delete specific partition table entries only
# This is more realistic as it simulates accidental deletion
# hexedit test_disk.dd  # Manually edit bytes 446-509 to zero out partition entries

# Verify partition table is gone
sudo fdisk -l test_disk.dd

# Document partition deletion simulation
echo "=== Partition Deletion Simulation ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Action: Zeroed partition table (first 512 bytes)" >> ../logs/lab_creation.log
echo "Original (with partition): test_disk_with_partition.dd" >> ../logs/lab_creation.log
echo "Damaged (no partition table): test_disk.dd" >> ../logs/lab_creation.log
```

##### Step 8: Detach Loop Device and Prepare Evidence
```bash
# Detach the loop device
sudo losetup -d /dev/loop0

# Rename the damaged image to match lab expectations
mv test_disk.dd deleted_partition.dd

# Keep the original with partition table for comparison
mv test_disk_with_partition.dd original_with_partition.dd

# Calculate final hashes for both images
echo "=== Final Image Hashes ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log
echo "Damaged image (deleted_partition.dd):" >> ../logs/lab_creation.log
md5sum deleted_partition.dd >> ../logs/lab_creation.log
sha1sum deleted_partition.dd >> ../logs/lab_creation.log
echo "Original image (original_with_partition.dd):" >> ../logs/lab_creation.log
md5sum original_with_partition.dd >> ../logs/lab_creation.log
sha1sum original_with_partition.dd >> ../logs/lab_creation.log

# Display lab creation summary
echo ""
echo "=== LAB CREATION COMPLETE ==="
echo "Created files:"
echo "- deleted_partition.dd (damaged image for recovery practice)"
echo "- original_with_partition.dd (backup with intact partition table)"
echo "- logs/lab_creation.log (detailed creation log)"
echo ""
echo "You can now proceed with the partition recovery exercises using deleted_partition.dd"
echo "The original_with_partition.dd file serves as a reference for verification"
```

#### Lab Environment Verification
```bash
# Verify the lab environment is ready
echo "=== Lab Environment Verification ===" >> ../logs/lab_creation.log
echo "Date: $(date)" >> ../logs/lab_creation.log

# Check file sizes
ls -lh deleted_partition.dd original_with_partition.dd >> ../logs/lab_creation.log

# Verify the damaged image has no partition table
echo "Partition table check on damaged image:" >> ../logs/lab_creation.log
sudo fdisk -l deleted_partition.dd >> ../logs/lab_creation.log 2>&1

# Verify the original image has intact partition table
echo "Partition table check on original image:" >> ../logs/lab_creation.log
sudo fdisk -l original_with_partition.dd >> ../logs/lab_creation.log 2>&1

echo "Lab environment creation and verification complete!"
```

### Verify Lab Image Integrity
```bash
# Navigate to evidence directory
cd ~/forensics_lab/partition_recovery/evidence

# Calculate initial hash values for chain of custody
echo "=== Initial Evidence Hashes ===" > ../logs/evidence_hashes.log
echo "File: deleted_partition.dd (damaged image)" >> ../logs/evidence_hashes.log
echo "Date: $(date)" >> ../logs/evidence_hashes.log
md5sum deleted_partition.dd >> ../logs/evidence_hashes.log
sha1sum deleted_partition.dd >> ../logs/evidence_hashes.log
sha256sum deleted_partition.dd >> ../logs/evidence_hashes.log
echo "" >> ../logs/evidence_hashes.log

# Also hash the reference image
echo "File: original_with_partition.dd (reference image)" >> ../logs/evidence_hashes.log
echo "Date: $(date)" >> ../logs/evidence_hashes.log
md5sum original_with_partition.dd >> ../logs/evidence_hashes.log
sha1sum original_with_partition.dd >> ../logs/evidence_hashes.log
sha256sum original_with_partition.dd >> ../logs/evidence_hashes.log
echo "" >> ../logs/evidence_hashes.log

# Display file information
echo "=== File Information ===" >> ../logs/evidence_hashes.log
echo "Damaged image:" >> ../logs/evidence_hashes.log
file deleted_partition.dd >> ../logs/evidence_hashes.log
ls -la deleted_partition.dd >> ../logs/evidence_hashes.log
echo "Reference image:" >> ../logs/evidence_hashes.log
file original_with_partition.dd >> ../logs/evidence_hashes.log
ls -la original_with_partition.dd >> ../logs/evidence_hashes.log

# Verify partition table status
echo "=== Partition Table Status ===" >> ../logs/evidence_hashes.log
echo "Damaged image partition table:" >> ../logs/evidence_hashes.log
sudo fdisk -l deleted_partition.dd >> ../logs/evidence_hashes.log 2>&1
echo "Reference image partition table:" >> ../logs/evidence_hashes.log
sudo fdisk -l original_with_partition.dd >> ../logs/evidence_hashes.log 2>&1
```

---

## Phase 1: Initial Analysis and Preparation

### Examine the Disk Image
```bash
# Navigate to evidence directory
cd ~/forensics_lab/partition_recovery/evidence

# Examine file structure
file deleted_partition.dd

# Check disk image size and properties
ls -lh deleted_partition.dd

# Use hexdump to examine first few sectors
hexdump -C deleted_partition.dd | head -20

# Look for partition signatures
strings deleted_partition.dd | head -50
```

### Create Working Copy (Forensic Best Practice)
```bash
# Create working copy to preserve original evidence
cp deleted_partition.dd ../original_images/deleted_partition_original.dd

# Create analysis copy
cp deleted_partition.dd ../recovered_partitions/deleted_partition_working.dd

# Document the copying process
echo "=== Working Copy Created ===" >> ../logs/analysis_log.log
echo "Date: $(date)" >> ../logs/analysis_log.log
echo "Original preserved in: original_images/" >> ../logs/analysis_log.log
echo "Working copy created in: recovered_partitions/" >> ../logs/analysis_log.log
```

---

## Phase 2: Partition Recovery with TestDisk

### Launch TestDisk
```bash
# Navigate to working directory
cd ~/forensics_lab/partition_recovery/recovered_partitions

# Launch TestDisk with working copy
sudo testdisk deleted_partition_working.dd
```

### TestDisk Interactive Process

#### Step 1: Select Disk
```
TestDisk Interface:
1. Select [Create] for new log file
2. Press Enter to continue
3. Select the disk image: deleted_partition_working.dd
4. Press Enter to proceed
```

#### Step 2: Choose Partition Table Type
```
Partition Table Types:
- [Intel] for PC/Intel partition tables (MBR)
- [EFI GPT] for GPT partition tables
- [Mac] for Apple partition tables
- [None] for no partition table

Most common: Select [Intel] and press Enter
```

#### Step 3: Analyze Partition Structure
```
Main Menu Options:
1. Select [Analyse] to examine current partition structure
2. Press Enter to proceed
3. TestDisk will display current partition table
4. Look for deleted or missing partitions
```

#### Step 4: Quick Search for Partitions
```
Analyse Menu:
1. Select [Quick Search] to find deleted partitions
2. Press Enter to start search
3. TestDisk will scan for partition signatures
4. Review found partitions:
   - D = Deleted partition
   - P = Primary partition
   - L = Logical partition
   - * = Bootable partition
```

#### Step 5: Deep Search (if needed)
```
If Quick Search doesn't find partitions:
1. Press 'Enter' to continue to main menu
2. Select [Deeper Search] for comprehensive scan
3. Wait for scan completion (may take time)
4. Review all found partitions
```

#### Step 6: Select Partitions to Recover
```
Partition Selection:
1. Use arrow keys to navigate partitions
2. Press 'P' to list files in a partition (preview)
3. Use 'Enter' to select/deselect partitions
4. Ensure correct partitions are marked for recovery
5. Press 'Enter' when selection is complete
```

#### Step 7: Write Partition Table
```
Final Step:
1. Select [Write] to restore partition table
2. Confirm by typing 'Y' when prompted
3. TestDisk will write new partition table
4. Press Enter to complete process
5. Quit TestDisk
```

### Command Line Documentation
```bash
# Document TestDisk results
echo "=== TestDisk Partition Recovery ===" >> ../logs/analysis_log.log
echo "Date: $(date)" >> ../logs/analysis_log.log
echo "Tool: TestDisk" >> ../logs/analysis_log.log
echo "Action: Partition table recovery" >> ../logs/analysis_log.log

# Check if partitions are now accessible
sudo fdisk -l deleted_partition_working.dd

# Create loop device for mounting
sudo losetup -P /dev/loop0 deleted_partition_working.dd

# List available partitions
lsblk /dev/loop0

# Document partition information
sudo fdisk -l /dev/loop0 >> ../logs/analysis_log.log
```

---

## Phase 3: Mounting and Accessing Recovered Partitions

### Mount Recovered Partitions
```bash
# Create mount points
sudo mkdir -p /mnt/recovered_partition1
sudo mkdir -p /mnt/recovered_partition2

# Mount first partition (adjust partition number as needed)
sudo mount /dev/loop0p1 /mnt/recovered_partition1

# If multiple partitions exist
# sudo mount /dev/loop0p2 /mnt/recovered_partition2

# Verify mount success
df -h | grep recovered

# Document mount information
echo "=== Partition Mounting ===" >> ../logs/analysis_log.log
mount | grep recovered >> ../logs/analysis_log.log
```

### Explore Recovered Files
```bash
# List files in recovered partition
ls -la /mnt/recovered_partition1/

# Create detailed file listing
find /mnt/recovered_partition1/ -type f -exec ls -la {} \; > ../logs/recovered_files_list.txt

# Count recovered files
echo "Total files recovered: $(find /mnt/recovered_partition1/ -type f | wc -l)" >> ../logs/analysis_log.log

# Look for specific file types
echo "=== File Type Analysis ===" >> ../logs/analysis_log.log
find /mnt/recovered_partition1/ -name "*.txt" | wc -l | xargs echo "Text files:" >> ../logs/analysis_log.log
find /mnt/recovered_partition1/ -name "*.jpg" -o -name "*.jpeg" | wc -l | xargs echo "JPEG images:" >> ../logs/analysis_log.log
find /mnt/recovered_partition1/ -name "*.pdf" | wc -l | xargs echo "PDF files:" >> ../logs/analysis_log.log
find /mnt/recovered_partition1/ -name "*.doc" -o -name "*.docx" | wc -l | xargs echo "Word documents:" >> ../logs/analysis_log.log
```

### Copy Recovered Files for Analysis
```bash
# Create directory for recovered file copies
mkdir -p ../recovered_partitions/files_from_partition

# Copy all recovered files (preserving structure)
sudo cp -a /mnt/recovered_partition1/* ../recovered_partitions/files_from_partition/

# Change ownership to current user
sudo chown -R $(whoami):$(whoami) ../recovered_partitions/files_from_partition/

# Calculate hashes of recovered files
find ../recovered_partitions/files_from_partition/ -type f -exec md5sum {} \; > ../logs/recovered_files_hashes.md5
find ../recovered_partitions/files_from_partition/ -type f -exec sha1sum {} \; > ../logs/recovered_files_hashes.sha1

### Validation Against Original Image (Lab Verification)
```bash
# Since we created our own lab, we can validate our recovery against the original
cd ~/forensics_lab/partition_recovery/evidence

# Mount the original image to compare with recovered files
# IMPORTANT: Use -P flag to make partitions available
sudo losetup -P /dev/loop1 original_with_partition.dd

# Verify partitions are available
lsblk /dev/loop1
ls -la /dev/loop1*

# Create mount point and mount the original partition
sudo mkdir -p /mnt/original_partition
sudo mount /dev/loop1p1 /mnt/original_partition

# If you still get "Can't lookup blockdev" error, try:
# sudo partprobe /dev/loop1
# or detach and reattach with -P flag:
# sudo losetup -d /dev/loop1
# sudo losetup -P /dev/loop1 original_with_partition.dd

# Compare file structures
echo "=== Recovery Validation ===" >> ../logs/analysis_log.log
echo "Date: $(date)" >> ../logs/analysis_log.log

# Compare directory structures
echo "Original directory structure:" >> ../logs/analysis_log.log
find /mnt/original_partition -type f | sort >> ../logs/analysis_log.log
echo "Recovered directory structure:" >> ../logs/analysis_log.log
find /mnt/recovered_partition1 -type f | sort >> ../logs/analysis_log.log

# Compare file hashes
echo "=== File Hash Comparison ===" >> ../logs/analysis_log.log
echo "Original file hashes:" >> ../logs/analysis_log.log
find /mnt/original_partition -type f -exec md5sum {} \; | sort >> ../logs/analysis_log.log
echo "Recovered file hashes:" >> ../logs/analysis_log.log
find /mnt/recovered_partition1 -type f -exec md5sum {} \; | sort >> ../logs/analysis_log.log

# Generate comparison report
echo "=== Recovery Success Rate ===" >> ../logs/analysis_log.log
original_count=$(find /mnt/original_partition -type f | wc -l)
recovered_count=$(find /mnt/recovered_partition1 -type f | wc -l)
echo "Original files: $original_count" >> ../logs/analysis_log.log
echo "Recovered files: $recovered_count" >> ../logs/analysis_log.log
if [ $original_count -eq $recovered_count ]; then
    echo "SUCCESS: All files recovered!" >> ../logs/analysis_log.log
else
    echo "PARTIAL: $recovered_count of $original_count files recovered" >> ../logs/analysis_log.log
fi

# Cleanup validation mount
sudo umount /mnt/original_partition
sudo rmdir /mnt/original_partition
sudo losetup -d /dev/loop1
```
```

---

## Phase 4: File Carving for Missing Files

### Unmount Partitions (if mounted)
```bash
# Safely unmount partitions before carving
sudo umount /mnt/recovered_partition1
# sudo umount /mnt/recovered_partition2  # if multiple partitions

# Remove loop device
sudo losetup -d /dev/loop0
```

### File Carving with Foremost

#### Configure Foremost
```bash
# Create Foremost configuration directory
mkdir -p ~/forensics_lab/partition_recovery/config

# Copy default Foremost configuration
sudo cp /etc/foremost.conf ~/forensics_lab/partition_recovery/config/foremost.conf

# Make configuration editable
chmod 644 ~/forensics_lab/partition_recovery/config/foremost.conf

# Edit configuration to enable specific file types
nano ~/forensics_lab/partition_recovery/config/foremost.conf

# Common file types to enable (uncomment these lines):
# jpg     y       4500000         \xff\xd8\xff\xe0\x00\x10        \xff\xd9
# gif     y       20000000        \x47\x49\x46\x38\x37\x61        \x00\x3b
# gif     y       20000000        \x47\x49\x46\x38\x39\x61        \x00\x3b
# png     y       20000000        \x50\x4e\x47?                   \xff\xfc\xfd\xfe
# pdf     y       5000000         \x25\x50\x44\x46                \x0a\x25\x25\x45\x4f\x46
# zip     y       20000000        \x50\x4b\x03\x04
```

#### Run Foremost File Carving
```bash
# Navigate to carved files directory
cd ~/forensics_lab/partition_recovery/carved_files

# Run Foremost on the entire disk image
foremost -c ../config/foremost.conf -i ../recovered_partitions/deleted_partition_working.dd -o foremost_output

# Alternative: Use default configuration
# foremost -i ../recovered_partitions/deleted_partition_working.dd -o foremost_output_default

# Document carving process
echo "=== File Carving with Foremost ===" >> ../logs/analysis_log.log
echo "Date: $(date)" >> ../logs/analysis_log.log
echo "Input: deleted_partition_working.dd" >> ../logs/analysis_log.log
echo "Output: carved_files/foremost_output/" >> ../logs/analysis_log.log
```

#### Analyze Carved Files
```bash
# Navigate to Foremost output
cd foremost_output

# Review audit file
cat audit.txt

# List carved file directories
ls -la

# Count carved files by type
echo "=== Carved Files Summary ===" >> ../../logs/analysis_log.log
for dir in */; do
    if [ -d "$dir" ]; then
        count=$(find "$dir" -type f | wc -l)
        echo "$dir: $count files" >> ../../logs/analysis_log.log
    fi
done

# Create detailed inventory of carved files
find . -type f -exec ls -la {} \; > ../../logs/carved_files_inventory.txt
```

### File Carving with Scalpel (Alternative Method)

#### Configure Scalpel
```bash
# Create Scalpel configuration
cd ~/forensics_lab/partition_recovery/config

# Copy default Scalpel configuration
sudo cp /etc/scalpel/scalpel.conf ./scalpel.conf
chmod 644 scalpel.conf

# Edit configuration for specific file types
nano scalpel.conf

# Example configurations to uncomment:
# jpg     y       4500000         \xff\xd8\xff\xe0\x00\x10JFIF     \xff\xd9
# png     y       20000000        \x50\x4e\x47                     \x49\x45\x4e\x44\xae\x42\x60\x82
# pdf     y       5000000         \x25\x50\x44\x46                 \x0a\x25\x25\x45\x4f\x46\x0a
```

#### Run Scalpel
```bash
# Navigate to carved files directory
cd ~/forensics_lab/partition_recovery/carved_files

# Run Scalpel
scalpel -c ../config/scalpel.conf -o scalpel_output ../recovered_partitions/deleted_partition_working.dd

# Review results
cd scalpel_output
cat scalpel-output.txt

# Document Scalpel results
echo "=== File Carving with Scalpel ===" >> ../../logs/analysis_log.log
echo "Date: $(date)" >> ../../logs/analysis_log.log
cat scalpel-output.txt >> ../../logs/analysis_log.log
```

---

## Phase 5: File Analysis and Verification

### Verify File Integrity
```bash
# Navigate to carved files
cd ~/forensics_lab/partition_recovery/carved_files

# Check file types using 'file' command
echo "=== File Type Verification ===" > ../logs/file_analysis.log
find foremost_output/ -type f -exec file {} \; >> ../logs/file_analysis.log

# Identify complete vs. partial files
echo "=== File Completeness Analysis ===" >> ../logs/file_analysis.log

# For JPEG files - check for proper headers and footers
find foremost_output/jpg/ -name "*.jpg" -exec sh -c '
    echo "Analyzing: $1"
    echo "Header check:"
    hexdump -C "$1" | head -1
    echo "Footer check:"
    hexdump -C "$1" | tail -1
    echo "---"
' _ {} \; >> ../logs/file_analysis.log 2>/dev/null

# For PDF files - check structure
find foremost_output/pdf/ -name "*.pdf" -exec sh -c '
    echo "PDF Analysis: $1"
    strings "$1" | grep -E "(PDF|EOF)" | head -5
    echo "---"
' _ {} \; >> ../logs/file_analysis.log 2>/dev/null
```

### Calculate File Hashes
```bash
# Calculate hashes for all carved files
echo "=== Carved Files Hashes ===" > ../logs/carved_files_hashes.log

# MD5 hashes
find foremost_output/ -type f -exec md5sum {} \; > ../logs/carved_files_md5.txt

# SHA1 hashes
find foremost_output/ -type f -exec sha1sum {} \; > ../logs/carved_files_sha1.txt

# SHA256 hashes (for higher security)
find foremost_output/ -type f -exec sha256sum {} \; > ../logs/carved_files_sha256.txt

# Combine hash information
echo "Date: $(date)" >> ../logs/carved_files_hashes.log
echo "MD5 Hashes:" >> ../logs/carved_files_hashes.log
cat ../logs/carved_files_md5.txt >> ../logs/carved_files_hashes.log
echo "" >> ../logs/carved_files_hashes.log
echo "SHA1 Hashes:" >> ../logs/carved_files_hashes.log
cat ../logs/carved_files_sha1.txt >> ../logs/carved_files_hashes.log
```

### Compare with Known Hash Databases
```bash
# If you have known hash databases (like NSRL)
# Example comparison (adapt path to your hash database)
# comm -12 <(sort ../logs/carved_files_md5.txt) <(sort /path/to/nsrl/hashes.txt) > known_files.txt

# Create custom known good/bad hash database for this case
echo "=== Hash Database Comparison ===" >> ../logs/file_analysis.log
echo "Creating case-specific hash database..." >> ../logs/file_analysis.log

# Document any matches with known file hashes
# This would be customized based on your specific investigation needs
```

---

## Phase 6: Documentation and Reporting

### Create Case Summary
```bash
# Create comprehensive case report
cd ~/forensics_lab/partition_recovery/reports

cat > case_summary.md << 'EOF'
# Digital Forensics Case Report: Partition Recovery and File Carving

## Case Information
- Case Number: [Enter case number]
- Investigator: [Your name]
- Date: [Current date]
- Evidence: deleted_partition.dd

## Investigation Summary
### Objectives
- Recover deleted partition structure
- Access files within recovered partition
- Perform file carving for additional file recovery
- Document all findings with forensic integrity

### Tools Used
- TestDisk: Partition recovery
- Foremost: File carving
- Scalpel: Alternative file carving
- Standard Linux utilities: File analysis and hashing

## Findings
[To be filled based on actual results]

### Partition Recovery Results
- Number of partitions recovered: [X]
- File system types: [NTFS/FAT32/ext4/etc.]
- Total files recovered from partition: [X]

### File Carving Results
- Total files carved: [X]
- File types recovered: [List types]
- Complete files: [X]
- Partial files: [X]

## Evidence Integrity
- Original evidence hash maintained
- Working copies used for analysis
- Chain of custody documented

## Conclusions
[Your analysis and conclusions]

## Appendices
- A: File listings
- B: Hash values
- C: Tool output logs
- D: Technical configuration details
EOF

# Fill in actual values from your analysis
nano case_summary.md
```

### Generate Final Report Package
```bash
# Create final report structure
mkdir -p final_report/{evidence_hashes,recovered_files,carved_files,logs,configs}

# Copy essential files to report package
cp ../logs/evidence_hashes.log final_report/evidence_hashes/
cp ../logs/analysis_log.log final_report/logs/
cp ../logs/recovered_files_hashes.* final_report/recovered_files/
cp ../logs/carved_files_hashes.log final_report/carved_files/
cp ../config/* final_report/configs/

# Create archive of complete findings
tar -czf partition_recovery_case_$(date +%Y%m%d_%H%M%S).tar.gz final_report/

# Document archive creation
echo "=== Final Report Archive ===" >> ../logs/analysis_log.log
echo "Date: $(date)" >> ../logs/analysis_log.log
echo "Archive: partition_recovery_case_$(date +%Y%m%d_%H%M%S).tar.gz" >> ../logs/analysis_log.log
ls -la *.tar.gz >> ../logs/analysis_log.log
```

---

## Phase 7: Cleanup and Evidence Preservation

### Secure Cleanup
```bash
# Navigate to lab directory
cd ~/forensics_lab/partition_recovery

# Safely unmount any remaining mounted filesystems
sudo umount /mnt/recovered_partition* 2>/dev/null
sudo rmdir /mnt/recovered_partition* 2>/dev/null

# Remove any remaining loop devices
sudo losetup -D

# Secure deletion of working copies (if required)
# WARNING: Only do this if you have preserved the original evidence
# shred -vfz -n 3 recovered_partitions/deleted_partition_working.dd

# Document cleanup
echo "=== Case Cleanup ===" >> logs/analysis_log.log
echo "Date: $(date)" >> logs/analysis_log.log
echo "Working copies cleaned up" >> logs/analysis_log.log
echo "Original evidence preserved" >> logs/analysis_log.log
```

### Verification of Evidence Preservation
```bash
# Verify original evidence integrity
cd evidence/
echo "=== Final Evidence Verification ===" >> ../logs/analysis_log.log
echo "Date: $(date)" >> ../logs/analysis_log.log
md5sum deleted_partition.dd >> ../logs/analysis_log.log
sha1sum deleted_partition.dd >> ../logs/analysis_log.log

# Compare with initial hashes
echo "Comparing with initial evidence hashes..." >> ../logs/analysis_log.log
diff <(grep deleted_partition.dd ../logs/evidence_hashes.log) <(md5sum deleted_partition.dd) >> ../logs/analysis_log.log
```

---

## Troubleshooting Common Issues

### TestDisk Issues
```bash
# If TestDisk cannot find partitions:
# 1. Try different partition table types
# 2. Use "Deeper Search" option
# 3. Check disk image integrity

# If permissions issues:
sudo chown $(whoami):$(whoami) deleted_partition_working.dd
chmod 644 deleted_partition_working.dd

# If loop device issues:
sudo losetup -D  # Clear all loop devices
```

### Mounting Issues
```bash
# If partition won't mount:
# Check filesystem type
sudo file -s /dev/loop0p1

# Try read-only mount
sudo mount -o ro /dev/loop0p1 /mnt/recovered_partition1

# For corrupted filesystems, try:
sudo fsck -f /dev/loop0p1  # Use with caution!
```

### File Carving Issues
```bash
# If Foremost produces no results:
# 1. Check configuration file
# 2. Verify file signatures are enabled
# 3. Try with different block sizes

# Increase verbosity for debugging
foremost -v -i input.dd -o output_dir

# If insufficient disk space:
# Monitor space usage
df -h
# Clean up previous carved files if safe to do so
```

---

## Advanced Techniques (Optional)

### Using PhotoRec for Additional File Recovery
```bash
# PhotoRec is included with TestDisk
photorec /log deleted_partition_working.dd

# PhotoRec will launch interactive interface:
# 1. Select disk
# 2. Select partition type
# 3. Select file types to recover
# 4. Choose output directory
# 5. Start recovery process
```

### Hexadecimal Analysis
```bash
# Use hex editor for manual analysis
hexedit deleted_partition_working.dd

# Or use xxd for command-line hex viewing
xxd deleted_partition_working.dd | head -50

# Search for specific hex patterns
xxd deleted_partition_working.dd | grep -i "ffd8"  # JPEG headers
```

### Automated Analysis Script
```bash
# Create analysis automation script
cat > analyze_partition.sh << 'EOF'
#!/bin/bash

# Automated partition analysis script
IMAGE_FILE="$1"
OUTPUT_DIR="$2"

if [ $# -ne 2 ]; then
    echo "Usage: $0 <image_file> <output_directory>"
    exit 1
fi

echo "Starting automated analysis of $IMAGE_FILE"

# Create output structure
mkdir -p "$OUTPUT_DIR"/{logs,recovered,carved}

# Calculate initial hashes
echo "Calculating evidence hashes..."
md5sum "$IMAGE_FILE" > "$OUTPUT_DIR/logs/evidence.md5"
sha1sum "$IMAGE_FILE" > "$OUTPUT_DIR/logs/evidence.sha1"

# Run file carving
echo "Running Foremost file carving..."
foremost -i "$IMAGE_FILE" -o "$OUTPUT_DIR/carved/foremost_output"

# Generate report
echo "Generating analysis report..."
echo "Analysis completed: $(date)" > "$OUTPUT_DIR/logs/analysis_complete.log"
find "$OUTPUT_DIR/carved" -type f | wc -l | xargs echo "Total carved files:" >> "$OUTPUT_DIR/logs/analysis_complete.log"

echo "Analysis complete. Results in: $OUTPUT_DIR"
EOF

chmod +x analyze_partition.sh

# Use the script
# ./analyze_partition.sh deleted_partition_working.dd automated_analysis_output
```

---

## Summary 


### Best Practices Reinforced
- Always work with copies, never original evidence
- Calculate and verify hashes at every step
- Document all actions and findings thoroughly
- Use multiple tools for verification and validation
- Maintain proper chain of custody
- Create controlled test environments for skill development
- Validate recovery techniques against known good data

### Enhanced Learning Through Lab Creation
- **Deeper Understanding**: Creating the partition structure provides insight into how data is organized
- **Real-world Simulation**: The lab mimics actual forensic scenarios where partition tables are corrupted or deleted
- **Validation Capability**: Having the original allows verification of recovery techniques
- **Repeatable Process**: Students can create multiple scenarios with different file systems and content
- **Error Analysis**: Students can experiment with different types of partition damage

### Next Steps
- Practice with different file systems (NTFS, FAT32, ext3, HFS+)
- Experiment with GPT partition tables instead of MBR
- Learn advanced carving techniques and custom file signatures
- Study partition table structures in detail across different operating systems
- Explore automated forensic frameworks and their partition recovery capabilities
- Practice testimony and report writing for court proceedings
- Create more complex scenarios with multiple partitions and file systems

---

## Additional Resources

### Documentation and References
- TestDisk documentation: https://www.cgsecurity.org/wiki/TestDisk
- Foremost manual: `man foremost`
- Digital Forensics Guide: https://www.sans.org/blog/digital-forensics-books/
- Linux forensics resources: Various SANS and NIST publications

### Practice Datasets
- Digital Corpora: https://digitalcorpora.org/
- NIST Computer Forensics Reference Data Sets
- Forensics Wiki test images

### Professional Development
- SANS FOR500: Windows Forensic Analysis
- SANS FOR508: Advanced Incident Response, Threat Hunting, and Digital Forensics
- EnCase Certified Examiner (EnCE)
- Certified Computer Hacker Forensic Investigator (CHFI)

---

