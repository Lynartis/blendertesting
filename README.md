
//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint   uImpulses         = 0;

    float2 vTotalOffset      = float2( 0.0f, 0.0f );
    float  fTotalWeight      = 0.0f;

    float  fCurrentMaxRadius = 0.0f;
    float  fPrevRadius       = 0.0f;
    float  fMaxRadius        = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld =
        WORLD_PIVOT +
        ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir =
                vWorld -
                GET_IMPULSE_POS( j );


            float fSqrDist =
                1.0f -
                saturate(
                    dot( vDir, vDir ) *
                    GET_IMPULSE_RCP_RADIUS( j )
                );


            if( fSqrDist > 0.0f )
            {
                float fRadius =
                    sqrt(
                        1.0f /
                        GET_IMPULSE_RCP_RADIUS( j )
                    );


                // Weighted XY offset from impulse center
                vTotalOffset +=
                    vDir *
                    fSqrDist;

                fTotalWeight +=
                    fSqrDist;


                // Keep the biggest CURRENT radius
                fCurrentMaxRadius =
                    max(
                        fCurrentMaxRadius,
                        fRadius
                    );


                uImpulses++;
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev =
        texHistory.Sample(
            samplerHistory,
            i.vTex0 - UV_DISPLACEMENT
        );


    float fPrevPower =
        saturate(
            TrailMap_GetPower( vPrev ) -
            DECAY
        );


    // --------------------------------------------------------
    // PREVIOUS HISTORY
    // --------------------------------------------------------

    if( fPrevPower > 0.0f )
    {
        // Decode previous signed XY offset
        float2 vPrevOffset =
            (vPrev.rg * 2.0f - 1.0f) *
            MAX_IMPULSE_RADIUS;


        // Decode previous radius
        fPrevRadius =
            vPrev.b *
            MAX_IMPULSE_RADIUS;


        // Previous offset contributes according to
        // its remaining power
        vTotalOffset +=
            vPrevOffset *
            fPrevPower;

        fTotalWeight +=
            fPrevPower;
    }


    // --------------------------------------------------------
    // MAX RADIUS
    // --------------------------------------------------------

    // Small impulses cannot cut an existing bigger radius
    fMaxRadius =
        max(
            fCurrentMaxRadius,
            fPrevRadius
        );


    // --------------------------------------------------------
    // OUTPUT
    // --------------------------------------------------------

    float2 vRetOffset =
        float2( 0.5f, 0.5f );

    float fRetRadius = 0.0f;
    float fRetPower  = 0.0f;


    if( fTotalWeight > 0.0f )
    {
        float2 vOffset =
            vTotalOffset /
            fTotalWeight;


        // ----------------------------------------------------
        // RG
        //
        // Signed XY offset packed into 0..1
        //
        // -MAX radius -> 0
        //  0          -> 0.5
        // +MAX radius -> 1
        // ----------------------------------------------------

        vRetOffset =
            saturate(
                (vOffset / MAX_IMPULSE_RADIUS) *
                0.5f +
                0.5f
            );


        // ----------------------------------------------------
        // B
        //
        // Maximum radius packed into 0..1
        // ----------------------------------------------------

        fRetRadius =
            saturate(
                fMaxRadius /
                MAX_IMPULSE_RADIUS
            );


        // ----------------------------------------------------
        // A
        //
        // IMPORTANT:
        //
        // A current small impulse must NOT reactivate an old
        // larger sphere.
        //
        // Only refresh to 1 when the current radius is the
        // radius that wins.
        // ----------------------------------------------------

        if( uImpulses > 0 )
        {
            fRetPower =
                (fCurrentMaxRadius >= fPrevRadius)
                ? 1.0f
                : fPrevPower;
        }
        else
        {
            fRetPower =
                fPrevPower;
        }
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = XY offset from impulse center
    // B  = maximum impulse radius
    // A  = trail power / decay
    // --------------------------------------------------------

    float4 vRet =
        float4(
            0.0f,
            0.0f,
            0.0f,
            0.0f
        );


    vRet.rg = vRetOffset;
    vRet.b  = fRetRadius;
    vRet.a  = fRetPower;


    return vRet;
}
