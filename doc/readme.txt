package org.iq80.leveldb.impl;

import com.google.common.base.Preconditions;
import org.iq80.leveldb.util.ByteBufferSupport;
import org.iq80.leveldb.util.Closeables;
import org.iq80.leveldb.util.Slice;
import org.iq80.leveldb.util.SliceInput;
import org.iq80.leveldb.util.SliceOutput;
import org.iq80.leveldb.util.Slices;

import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.FileChannel.MapMode;
import java.util.concurrent.atomic.AtomicBoolean;

import static org.iq80.leveldb.impl.LogConstants.BLOCK_SIZE;
import static org.iq80.leveldb.impl.LogConstants.HEADER_SIZE;
import static org.iq80.leveldb.impl.Logs.getChunkChecksum;

public class MMapLogWriter
        implements LogWriter
{
    private static final int PAGE_SIZE = 1024 * 1024;

    private final File file;
    private final long fileNumber;
    private final FileChannel fileChannel;
    private final AtomicBoolean closed = new AtomicBoolean();
    private MappedByteBuffer mappedByteBuffer;
    private long fileOffset;
    /**
     * Current offset in the current block
     */
    private int blockOffset;

    public MMapLogWriter(File file, long fileNumber)
            throws IOException
    {
write breakpoint-->        Preconditions.checkNotNull(file, "file is null");
        Preconditions.checkArgument(fileNumber >= 0, "fileNumber is negative");
        this.file = file;
        
-------------------------------------------


    @Override
    public synchronized void addRecord(Slice record, boolean force)
            throws IOException
    {
        Preconditions.checkState(!closed.get(), "Log has been closed");

        SliceInput sliceInput = record.input();

        // used to track first, middle and last blocks
        boolean begin = true;

        // Fragment the record int chunks as necessary and write it.  Note that if record
        // is empty, we still want to iterate once to write a single
        // zero-length chunk.
        do {
            int bytesRemainingInBlock = BLOCK_SIZE - blockOffset;
            Preconditions.checkState(bytesRemainingInBlock >= 0);

            // Switch to a new block if necessary
            if (bytesRemainingInBlock < HEADER_SIZE) {
                if (bytesRemainingInBlock > 0) {
                    // Fill the rest of the block with zeros
                    // todo lame... need a better way to write zeros
                    ensureCapacity(bytesRemainingInBlock);
                    mappedByteBuffer.put(new byte[bytesRemainingInBlock]);
                }
                blockOffset = 0;
                bytesRemainingInBlock = BLOCK_SIZE - blockOffset;
            }

            // Invariant: we never leave less than HEADER_SIZE bytes available in a block
            int bytesAvailableInBlock = bytesRemainingInBlock - HEADER_SIZE;
            Preconditions.checkState(bytesAvailableInBlock >= 0);

            // if there are more bytes in the record then there are available in the block,
            // fragment the record; otherwise write to the end of the record
            boolean end;
            int fragmentLength;
            if (sliceInput.available() > bytesAvailableInBlock) {
                end = false;
                fragmentLength = bytesAvailableInBlock;
            }
            else {
                end = true;
                fragmentLength = sliceInput.available();
            }

            // determine block type
            LogChunkType type;
            if (begin && end) {
                type = LogChunkType.FULL;
            }
            else if (begin) {
                type = LogChunkType.FIRST;
            }
            else if (end) {
                type = LogChunkType.LAST;
            }
            else {
                type = LogChunkType.MIDDLE;
            }

            // write the chunk
            writeChunk(type, sliceInput.readBytes(fragmentLength));

            // we are no longer on the first chunk
            begin = false;
        } while (sliceInput.isReadable());

        if (force) {
MappedByteBuffer write to disk--->            mappedByteBuffer.force();
        }
        
           
      
------------------------------------------


            List<Long> logs = newArrayList();
            for (File filename : filenames) {
                FileInfo fileInfo = Filename.parseFileName(filename);

                if (fileInfo != null &&
                        fileInfo.getFileType() == FileType.LOG &&
                        ((fileInfo.getFileNumber() >= minLogNumber) || (fileInfo.getFileNumber() == previousLogNumber))) {
                    logs.add(fileInfo.getFileNumber());
                }
            }

            // Recover in the order in which the logs were generated
            VersionEdit edit = new VersionEdit();
            Collections.sort(logs);
            for (Long fileNumber : logs) {
from file to memmap--->                long maxSequence = recoverLogFile(fileNumber, edit);
                if (versions.getLastSequence() < maxSequence) {
                    versions.setLastSequence(maxSequence);
                }
            }
            
            
---------------------------------------


package org.iq80.leveldb.impl;

import com.google.common.base.Preconditions;
import com.google.common.collect.Iterators;
import com.google.common.collect.PeekingIterator;
import org.iq80.leveldb.util.InternalIterator;
import org.iq80.leveldb.util.Slice;

import java.util.Map.Entry;
import java.util.concurrent.ConcurrentSkipListMap;
import java.util.concurrent.atomic.AtomicLong;

import static org.iq80.leveldb.util.SizeOf.SIZE_OF_LONG;

public class MemTable
        implements SeekingIterable<InternalKey, Slice>
{
    private final ConcurrentSkipListMap<InternalKey, Slice> table;
    private final AtomicLong approximateMemoryUsage = new AtomicLong();

    public MemTable(InternalKeyComparator internalKeyComparator)
    {
        table = new ConcurrentSkipListMap<>(internalKeyComparator);
    }

    public boolean isEmpty()
    {
        return table.isEmpty();
    }

    public long approximateMemoryUsage()
    {
        return approximateMemoryUsage.get();
    }

breakpoint--->    public void add(long sequenceNumber, ValueType valueType, Slice key, Slice value)
    {
        Preconditions.checkNotNull(valueType, "valueType is null");
        Preconditions.checkNotNull(key, "key is null");
        Preconditions.checkNotNull(valueType, "valueType is null");

        InternalKey internalKey = new InternalKey(key, sequenceNumber, valueType);
        table.put(internalKey, value);

        approximateMemoryUsage.addAndGet(key.length() + SIZE_OF_LONG + value.length());
    }


-----------------------------------------------


    private long recoverLogFile(long fileNumber, VersionEdit edit)
            throws IOException
    {
        Preconditions.checkState(mutex.isHeldByCurrentThread());
        File file = new File(databaseDir, Filename.logFileName(fileNumber));
        try (FileInputStream fis = new FileInputStream(file);
                FileChannel channel = fis.getChannel()) {
            LogMonitor logMonitor = LogMonitors.logMonitor();
            LogReader logReader = new LogReader(channel, logMonitor, true, 0);

            // Log(options_.info_log, "Recovering log #%llu", (unsigned long long) log_number);

            // Read all the records and add to a memtable
            long maxSequence = 0;
            MemTable memTable = null;
read record from file-->            for (Slice record = logReader.readRecord(); record != null; record = logReader.readRecord()) {
                SliceInput sliceInput = record.input();
                // read header
                if (sliceInput.available() < 12) {
                    logMonitor.corruption(sliceInput.available(), "log record too small");
                    continue;
                }
                long sequenceBegin = sliceInput.readLong();
                int updateSize = sliceInput.readInt();

                // read entries
                WriteBatchImpl writeBatch = readWriteBatch(sliceInput, updateSize);

                // apply entries to memTable
                if (memTable == null) {
                    memTable = new MemTable(internalKeyComparator);
                }
write record to memmap-->                writeBatch.forEach(new InsertIntoHandler(memTable, sequenceBegin));
                
                
---------------------------------


