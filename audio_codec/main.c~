//需先配置mpg123,lame,sndfile库

/*
	mpg123_to_wav.c

	copyright 2007 by the mpg123 project - free software under the terms of the LGPL 2.1
	see COPYING and AUTHORS files in distribution or http://mpg123.org
	initially written by Nicholas Humfrey
*/

// ./main /media/ioa_audio/UUI/mp3_test_music/非洲鼓狂野非洲.mp3 fzgkyfz.wav


#include <stdio.h>
#include <strings.h>
#include <mpg123.h>
#include <sndfile.h>
#include <lame.h>

void usage()
{
	printf("Usage: mpg123_to_wav <input> <output> [s16|f32 [ <buffersize>]]\n");
	exit(99);
}

void cleanup(mpg123_handle *mh)
{
	/* It's really to late for error checks here;-) */
	mpg123_close(mh);
	mpg123_delete(mh);
	mpg123_exit();
}



int main(int argc, char *argv[])
{
	SNDFILE* sndfile = NULL;
	SF_INFO sfinfo;
	mpg123_handle *mh = NULL;
	struct mpg123_frameinfo bsi;
	unsigned char* buffer = NULL;
	size_t buffer_size = 0;
	size_t done = 0;
	int  channels = 0, encoding = 0;
	long rate = 0;
	int  err  = MPG123_OK;
	off_t samples = 0;

	if (argc<3) usage();
	printf( "Input file: %s\n", argv[1]);
	printf( "Output file: %s\n", argv[2]);

	err = mpg123_init();
	if(err != MPG123_OK || (mh = mpg123_new(NULL, &err)) == NULL)
	{
		fprintf(stderr, "Basic setup goes wrong: %s", mpg123_plain_strerror(err));
		cleanup(mh);
		return -1;
	}

	/* Simple hack to enable floating point output. */
	if(argc >= 4 && !strcmp(argv[3], "f32")) mpg123_param(mh, MPG123_ADD_FLAGS, MPG123_FORCE_FLOAT, 0.);

	    /* Let mpg123 work with the file, that excludes MPG123_NEED_MORE messages. */
	if(    mpg123_open(mh, argv[1]) != MPG123_OK
	    /* Peek into track and get first output format. */
	    || mpg123_getformat(mh, &rate, &channels, &encoding) != MPG123_OK )
	{
		fprintf( stderr, "Trouble with mpg123: %s\n", mpg123_strerror(mh) );
		cleanup(mh);
		return -1;
	}

	if(encoding != MPG123_ENC_SIGNED_16 && encoding != MPG123_ENC_FLOAT_32)
	{ /* Signed 16 is the default output format anyways; it would actually by only different if we forced it.
	     So this check is here just for this explanation. */
		cleanup(mh);
		fprintf(stderr, "Bad encoding: 0x%x!\n", encoding);
		return -2;
	}
	/* Ensure that this output format will not change (it could, when we allow it). */
	mpg123_format_none(mh);//configure a mpg123 handle to accept no output format at all
	mpg123_format(mh, rate, channels, encoding);//set the audio format support of a mpg123_handle in detail

	/** Get frame information about the MPEG audio bitstream and store it in a mpg123_frameinfo structure. */
	mpg123_info(mh,&bsi);
	printf("MP3 file bitrate %d kbps.\n",bsi.bitrate);

	bzero(&sfinfo, sizeof(sfinfo) );
	sfinfo.samplerate = rate;
	sfinfo.channels = channels;
	sfinfo.format = SF_FORMAT_WAV|(encoding == MPG123_ENC_SIGNED_16 ? SF_FORMAT_PCM_16 : SF_FORMAT_FLOAT);
	printf("Creating WAV with %i channels and %liHz.\n", channels, rate);

	sndfile = sf_open(argv[2], SFM_WRITE, &sfinfo);
	if(sndfile == NULL){ fprintf(stderr, "Cannot open output file!\n"); cleanup(mh); return -2; }

	/* Buffer could be almost any size here, mpg123_outblock() is just some recommendation.
	   Important, especially for sndfile writing, is that the size is a multiple of sample size. */
	buffer_size = argc >= 5 ? atol(argv[4]) : mpg123_outblock(mh);//the max size of one frame's decoded output with current settings
	buffer = malloc( buffer_size );

	do
	{
		sf_count_t more_samples;
		err = mpg123_read( mh, buffer, buffer_size, &done );//read from stream and decode up to outmemsize bytes,buffer:output buffer,buffer_size:maximum number of bytes to write; done:address to store the number of actually decoded bytes to returns
		more_samples = encoding == MPG123_ENC_SIGNED_16
			? sf_write_short(sndfile, (short*)buffer, done/sizeof(short))
			: sf_write_float(sndfile, (float*)buffer, done/sizeof(float));
		if(more_samples < 0 || more_samples*mpg123_encsize(encoding) != done)
		{
			fprintf(stderr, "Warning: Written number of samples does not match the byte count we got from libmpg123: %li != %li\n", (long)(more_samples*mpg123_encsize(encoding)), (long)done);
		}
		samples += more_samples;
		/* We are not in feeder mode, so MPG123_OK, MPG123_ERR and MPG123_NEW_FORMAT are the only possibilities.
		   We do not handle a new format, MPG123_DONE is the end... so abort on anything not MPG123_OK. */
	} while (err==MPG123_OK);

	if(err != MPG123_DONE)
	fprintf( stderr, "Warning: Decoding ended prematurely because: %s\n",
	         err == MPG123_ERR ? mpg123_strerror(mh) : mpg123_plain_strerror(err) );

	sf_close( sndfile );

	samples /= channels;
	printf("%li samples written.\n", (long)samples);
	cleanup(mh);

	//encode
	encode(argv[2], "newmp3.mp3",bsi.bitrate);

	return 0;
}

int encode(char *inpath, char *outpath,int input_bitrate) {
    int status = 0;
    lame_global_flags *gfp;
    int ret_code;
    FILE *infp;
    FILE *outfp;
    short *input_buffer;
    int input_samples;
    char *mp3_buffer;
    int mp3_bytes;
	//#define INBUFSIZE 4096
	//#define MP3BUFSIZE (int)(1.25 * INBUFSIZE) + 7200
	int INBUFSIZE=4096;
	int MP3BUFSIZE=(int)(1.25 * INBUFSIZE) + 7200;
    /* Initialize the library. */
    gfp = lame_init();
    if (gfp == NULL) {
        printf("lame_init returned NULL\n");
        status = -1;
        goto exit;
    }


    /* Open our input and output files. */
    infp = fopen(inpath, "rb");
    outfp = fopen(outpath, "wb");

   //跳过头模块
    //FILE *fp2;  //新的输出WAV
    File_inform(infp,gfp);
    lame_set_brate(gfp,input_bitrate);//设置比特率压缩倍数,默认128kbps,对应压缩比11
    /* Set the encoding parameters. */
    ret_code = lame_init_params(gfp);
    if (ret_code < 0) {
        printf("lame_init_params returned %d\n", ret_code);
        status = -1;
        goto close_lame;
    }

    /* Allocate some buffers. */
    input_buffer = (short*)malloc(INBUFSIZE*2);
    mp3_buffer = (char*)malloc(MP3BUFSIZE);

    /* Read from the input file, encode, and write to the output file. */
    do {
        input_samples = fread(input_buffer, 2, INBUFSIZE, infp);
        if (input_samples > 0) {
            mp3_bytes = lame_encode_buffer_interleaved(gfp,input_buffer,input_samples / 2,mp3_buffer,MP3BUFSIZE);
            if (mp3_bytes < 0) {
                printf("lame_encode_buffer_interleaved returned %d\n", mp3_bytes);
                status = -1;
                goto free_buffers;}
	    else if (mp3_bytes > 0) {
                fwrite(mp3_buffer, 1, mp3_bytes, outfp);
            }
        }
    } while (input_samples == INBUFSIZE);

    /* Flush the encoder of any remaining bytes. */
    mp3_bytes = lame_encode_flush(gfp, mp3_buffer,sizeof(mp3_buffer));
    if (mp3_bytes > 0) {
        printf("writing %d mp3 bytes\n", mp3_bytes);
        fwrite(mp3_buffer, 1, mp3_bytes, outfp);
    }

    /* Clean up. */
free_buffers:
    free(mp3_buffer);
    free(input_buffer);
    fclose(outfp);
    fclose(infp);


close_lame:
    lame_close(gfp);

exit:
    return status;
}


int File_inform(FILE *fp,lame_global_flags *gfp)
{
	char addtion;
	int i=0;
	unsigned char ch[60];  //用来存储wav文件的头信息
        int channel_num_wav;   //用来存储wav文件的通道数
	int sample_bits_wav;  //用来存储wav文件的每个样本的字长
	int sample_rate_wav;   //用来存储wav文件的采样率
	for(i=0;i<58;i++)
	{
		ch[i]=fgetc(fp);
	}

	//取得WAV格式
	channel_num_wav=(short)ch[22] | ((short)ch[23]<<8);
	sample_bits_wav=(short)ch[34] | ((short)ch[35]<<8);
	sample_rate_wav=(short)ch[24] | ((short)ch[25]<<8)| ((short)ch[26]<<16)| ((short)ch[27]<<24);

 	/*设置输出MP3格式*/
	lame_set_num_channels(gfp,channel_num_wav);
	lame_set_in_samplerate(gfp, sample_rate_wav); //设置输入WAV的采样率
        lame_set_out_samplerate(gfp, sample_rate_wav);//设置输出MP3的采样率
	//lame_set_brate(gfp,128);//设置比特率压缩倍数,默认128kbps,对应压缩比11
	printf("channel_num_wav=%d \n",channel_num_wav);
	printf("sample_rate_wav=%d \n",sample_rate_wav);
	printf("sample_bits_wav=%d \n",sample_bits_wav);
	//lame_set_mode(gfp,3);
	//lame_set_quality(gfp,2); /* 2-high 5=medium 7-low*/
	//输出附加信息
	if(ch[16]==18)  //若Format Chunk的size大小为18，则该模块的最后两个字节为附加信息,addtion=2
	{               //若为16，则无附加信息,addtion=0
		addtion = 2;
	}
	else
	{
		addtion = 0;
	}

	// 存在Fact Chunk部分，字头长度为58（含Fomart Chunk附加信息）
	if(ch[36+addtion]==102 && ch[37+addtion]==97 && ch[38+addtion]==99 && ch[39+addtion]==116)
	{
		//fwrite(ch, 56+addtion, 1, fp2);   //把字头信息写入目标文件fp2
		fseek(fp,56+addtion,0);//把fp指针移动到离文件开头100字节处
	}
	else                                   // 不存在Fact Chunk部分，字头长度为46（含Fomart Chunk附加信息）
	{
		//fwrite(ch, 44+addtion, 1, fp2);    //把字头信息写入目标文件fp2
		fseek(fp,44+addtion,0);//把fp指针移动到离文件开头100字节处
	}
	return 0;
}
